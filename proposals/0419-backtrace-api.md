# Swift Backtrace API

* Proposal: [SE-0419](0419-backtrace-api.md)
* Authors: [Alastair Houghton](https://github.com/al45tair)
* Review Manager: [Steve Canon](https://github.com/stephentyrone)
* Status: **Active Review (January 23 - February 6, 2024)**
* Implementation: Implemented on main, requires explicit `_Backtracing` import.
* Review: ([pitch](https://forums.swift.org/t/pitch-swift-backtracing-api/62741/20)) ([review](https://forums.swift.org/t/se-0419-swift-backtracing-api/69595))

## Introduction

This year we are improving the usability of Swift for command line and
server-side development by adding first-class support for backtraces
to Swift.

The backtrace support consists of two parts; the first is the actual
backtracing implementation, and the second is the new API surface in
the Swift standard library.  This proposal concerns the latter.

## Motivation

In addition to the runtime providing backtraces when programs crash or
terminate abnormally, it is often useful for testing frameworks and
sometimes even library or application code to capture details of the
call stack at a point in time.

This functionality is somewhat tricky to implement correctly and any
implementation tends, of necessity, to be non-portable.  Existing
third-party packages that provide backtrace support have various
downsides, including lack of support for tracing through async frames,
and add additional dependencies to client packages and applications.

## Proposed solution

We will add a `Backtrace` struct to the standard library, with methods
to capture a backtrace from the current location, and support for
symbolication and symbol demangling.  All of the backtracing types will
exist in a new `Runtime` module.

Note, importantly, that **the API presented here is not async-signal-safe**,
and **it is not an appropriate tool with which to build a general purpose
crash reporter**.  The intended use case for this functionality is the
programmatic capture of backtraces during normal execution.

## Detailed design

The `Backtrace` struct will capture an `Array` of `Frame` objects,
each of which will represent a stack frame or a `Task` activation
context.

```swift
/// Holds a backtrace.
public struct Backtrace: CustomStringConvertible, Sendable {
  /// The type of an address.
  ///
  /// This is used as an opaque type; if you have some Address, you
  /// can ask if it's NULL, and you can attempt to convert it to a
  /// FixedWidthInteger.
  ///
  /// This is intentionally _not_ a pointer, because you shouldn't be
  /// dereferencing them; they may refer to some other process, for
  /// example.
  public protocol Address: Comparable, Hashable,
                           LosslessStringConvertible,
                           ExpressibleByIntegerLiteral {
    static var bitWidth: Int { get }

    var isNull: Bool { get }
  }

  /// The unwind algorithm to use.
  public enum UnwindAlgorithm {
    /// Choose the most appropriate for the platform.
    case auto

    /// Use the fastest viable method.
    ///
    /// Typically this means walking the frame pointers.
    case fast

    /// Use the most precise available method.
    ///
    /// On Darwin and on ELF platforms, this will use EH unwind
    /// information.  On Windows, it will use Win32 API functions.
    case precise
  }

  /// Represents an individual frame in a backtrace.
  @frozen
  public enum Frame: CustomStringConvertible, Sendable {
    /// An accurate program counter.
    ///
    /// This might come from a signal handler, or an exception or some
    /// other situation in which we have captured the actual program counter.
    case programCounter(any Address)

    /// A return address.
    ///
    /// Corresponds to a call from a normal function.
    case returnAddress(any Address)

    /// An async resume point.
    ///
    /// Corresponds to an `await` in an async task.
    case asyncResumePoint(any Address)

    /// Indicates a discontinuity in the backtrace.
    ///
    /// This occurs when you set a limit and a minimum number of frames at
    /// the top.  For example, if you set a limit of 10 frames and a minimum
    /// of 4 top frames, but the backtrace generated 100 frames, you will see
    ///
    ///    0: frame 100 <----- bottom of call stack
    ///    1: frame 99
    ///    2: frame 98
    ///    3: frame 97
    ///    4: frame 96
    ///    5: ...       <----- omittedFrames(92)
    ///    6: frame 3
    ///    7: frame 2
    ///    8: frame 1
    ///    9: frame 0   <----- top of call stack
    ///
    /// Note that the limit *includes* the discontinuity.
    ///
    /// This is good for handling cases involving deep recursion.
    case omittedFrames(Int)

    /// Indicates a discontinuity of unknown length.
    ///
    /// This can only be present at the end of a backtrace; in other cases
    /// we will know how many frames we have omitted.  For instance,
    ///
    ///    0: frame 100 <----- bottom of call stack
    ///    1: frame 99
    ///    2: frame 98
    ///    3: frame 97
    ///    4: frame 96
    ///    5: ...       <----- truncated
    case truncated

    /// The original program counter, with no adjustment.
    ///
    /// The value returned from this property is undefined if the frame
    /// is a discontinuity.
    public var originalProgramCounter: any Address { get }

    /// The adjusted program counter to use for symbolication.
    ///
    /// The value returned from this property is undefined if the frame
    /// is a discontinuity.
    public var adjustedProgramCounter: any Address { get }

    /// A textual description of this frame.
    ///
    /// @param width    Specifies the width in digits of the address field.
    ///
    /// @returns        A string describing the frame.
    public func description(width: Int) -> String

    /// A textual description of this frame.
    public var description: String { get }
  }

  /// Represents an image loaded in the process's address space
  public class Image: CustomStringConvertible, Identifiable, Sendable {
    /// The name of the image (e.g. libswiftCore.dylib).
    public var name: String { get }

    /// The full path to the image (e.g. /usr/lib/swift/libswiftCore.dylib).
    public var path: String { get }

    /// The build ID of the image, as a byte array (note that the exact number
    /// of bytes may vary, and that some images may not have a build ID).
    public var buildID: [UInt8]? { get }

    /// The base address of the image.
    public var baseAddress: some Address { get }

    /// The end of the text segment in this image.
    public var endOfText: some Address { get }

    /// Provide a textual description of this Image.
    ///
    /// @param width    Specifies the width in digits of the address fields.
    ///
    /// @returns        A string describing the Image.
    public func description(width: Int) -> String

    /// Provide a textual description of an Image.
    public var description: String { get }
  }

  /// The architecture of the system that captured this backtrace.
  public var architecture: String

  /// A `Sequence` that returns `Frame` instances
  struct FrameSequence: Sequence {
    typealias Element = Frame

    struct Iterator: IteratorProtocol {
      mutating func next() -> Frame?
    }

    func makeIterator() -> Iterator
  }

  /// A `Sequence` of captured frame information.
  ///
  /// The underlying storage is intentionally not exposed, because there may
  /// be cases where it's desirable to use a more compact form (for instance
  /// delta compression).
  public var frames: FrameSequence { get }

  /// A list of captured images.
  ///
  /// Some backtracing algorithms may require this information, in which case
  /// it will be filled in by the `capture()` method.  Other algorithms may
  /// not, in which case it will be empty and you can capture an image list
  /// separately yourself using `captureImages()`.
  public var images: [Image]?

  /// Holds information about the shared cache.
  public struct SharedCacheInfo: Identifiable, Sendable {
    /// The UUID from the shared cache.
    public var uuid: [UInt8] { get }

    /// The base address of the shared cache.
    public var baseAddress: any Address { get }

    /// Says whether there is in fact a shared cache.
    public var noCache: Bool
  }

  /// Information about the shared cache.
  ///
  /// Holds information about the shared cache.  On Darwin only, this is
  /// required for symbolication.  On non-Darwin platforms it will always
  /// be `nil`.
  public var sharedCacheInfo: SharedCacheInfo?

  /// Capture a backtrace from the current program location.
  ///
  /// The `capture()` method itself will not be included in the backtrace;
  /// i.e. the first frame will be the one in which `capture()` was called,
  /// and its programCounter value will be the return address for the
  /// `capture()` method call.
  ///
  /// @param algorithm     Specifies which unwind mechanism to use.  If this
  ///                      is set to `.auto`, we will use the platform default.
  /// @param limit         The backtrace will include at most this number of
  ///                      frames; you can set this to `nil` to remove the
  ///                      limit completely if required.
  /// @param offset        Says how many frames to skip; this makes it easy to
  ///                      wrap this API without having to inline things and
  ///                      without including unnecessary frames in the backtrace.
  /// @param top           Sets the minimum number of frames to capture at the
  ///                      top of the stack.
  ///
  /// @returns A new `Backtrace` struct.
  @inline(never)
  public static func capture(algorithm: UnwindAlgorithm = .auto,
                             limit: Int? = 64,
                             offset: Int = 0,
                             top: Int = 16) throws -> Backtrace

  /// Capture a list of the images currently mapped into the calling
  /// process.
  ///
  /// @returns A list of `Image`s.
  public static func captureImages() -> [Image]

  /// Capture shared cache information.
  ///
  /// @returns A `SharedCacheInfo`.
  public static func captureSharedCacheInfo() -> SharedCacheInfo

  /// Return a symbolicated version of the backtrace.
  ///
  /// @param images Specifies the set of images to use for symbolication.
  ///               If `nil`, the function will look to see if the `Backtrace`
  ///               has already captured images.  If it has, those will be
  ///               used; otherwise we will capture images at this point.
  ///
  /// @param sharedCacheInfo  Provides information about the location and
  ///                         identity of the shared cache, if applicable.
  ///
  /// @param showInlineFrames If `true` and we know how on the platform we're
  ///                         running on, add virtual frames to show inline
  ///                         function calls.
  ///
  /// @param useSymbolCache   If the system we are on has a symbol cache,
  ///                         says whether or not to use it.
  ///
  /// @returns A new `SymbolicatedBacktrace`.
  public func symbolicated(with images: [Image]? = nil,
                           sharedCacheInfo: SharedCacheInfo? = nil,
                           showInlineFrames: Bool = true,
                           useSymbolCache: Bool = true)
    -> SymbolicatedBacktrace?

  /// Provide a textual version of the backtrace.
  public var description: String { get }
}
```

We allow `Address` to be converted to a `FixedWidthInteger` by means of an
extension on `FixedWidthInteger`:

```swift
extension FixedWidthInteger {
  /// Convert from a Backtrace.Address.
  ///
  /// This initializer will return nil if the address width is larger than the
  /// type you are attempting to convert into.
  ///
  /// @param address The `Address` to convert.
  init?(_ address: some Backtrace.Address)
}
```

_Symbolication_, by which we mean the process of looking up the symbols
associated with addresses in a backtrace, is in general an expensive
process, and for efficiency reasons is normally performed for a backtrace
as a whole, rather than for individual frames.  It therefore makes sense
to provide a separate `SymbolicatedBacktrace` type and to provide a
method on a `Backtrace`
to symbolicate.

```swift
/// A symbolicated backtrace
public struct SymbolicatedBacktrace: CustomStringConvertible {
  /// The `Backtrace` from which this was constructed
  public var backtrace: Backtrace

  /// Represents a location in source code.
  ///
  /// The information in this structure comes from compiler-generated
  /// debug information and may not correspond to the current state of
  /// the filesystem --- it might even hold a path that only works
  /// from an entirely different machine.
  public struct SourceLocation: CustomStringConvertible, Sendable {
    /// The path of the source file.
    var path: String

    /// The line number.
    var line: Int

    /// The column number.
    var column: Int

    /// Provide a textual description.
    public var description: String { get }
  }

  /// Represents an individual frame in the backtrace.
  public struct Frame: CustomStringConvertible {
    /// The captured frame from the `Backtrace`.
    public var captured: Backtrace.Frame

    /// The result of doing a symbol lookup for this frame.
    public var symbol: Symbol?

    /// If `true`, then this frame was inlined.
    public var inlined: Bool = false

    /// `true` if this frame represents a Swift runtime failure.
    public var isSwiftRuntimeFailure: Bool

    /// `true` if this frame represents a Swift thunk function.
    public var isSwiftThunk: Bool { get }

    /// `true` if this frame is a system frame.
    public var isSystem: Bool { get }

    /// A textual description of this frame.
    ///
    /// @param width   Specifies the width in digits of the address fields.
    ///
    /// @returns       A string describing this frame.
    public func description(width: Int) -> String

    /// A textual description of this frame.
    public var description: String { get }
  }

  /// Represents a symbol we've located
  public class Symbol: CustomStringConvertible {
    /// The image in which the symbol for this address is located.
    public var image: Backtrace.Image

    /// The raw symbol name, before demangling.
    public var rawName: String

    /// The demangled symbol name.
    public lazy var name: String = demangleRawName()

    /// The offset from the symbol.
    public var offset: Int

    /// The source location, if available.
    public var sourceLocation: SourceLocation?

    /// True if this symbol represents a Swift runtime failure.
    public var isSwiftRuntimeFailure: Bool { get }

    /// True if this symbol is a Swift thunk function.
    public var isSwiftThunk: Bool { get }

    /// True if this symbol represents a system function.
    ///
    /// For instance, the `start` function from `dyld` on macOS is a system
    /// function, and we don't need to display it under normal circumstances.
    public var isSystem: Bool { get }

    /// Construct a new Symbol.
    public init(image: Backtrace.Image, rawName: String, offset: Int,
                sourceLocation: SourceLocation?)

    /// A textual description of this symbol.
    public var description: String { get }
  }

  /// A `Sequence` that returns `Frame` instances
  struct FrameSequence: Sequence {
    typealias Element = Frame

    struct Iterator: IteratorProtocol {
      mutating func next() -> Frame?
    }

    func makeIterator() -> Iterator
  }

  /// A list of captured frame information.
  public var frames: FrameSequence { get }

  /// A list of images found in the process.
  public var images: [Backtrace.Image]

  /// Shared cache information.
  public var sharedCacheInfo: Backtrace.SharedCacheInfo?

  /// True if this backtrace is a Swift runtime failure.
  public var isSwiftRuntimeFailure: Bool { get }

  /// Provide a textual version of the backtrace.
  public var description: String { get }
}
```

Example usage:

```swift
import Runtime

var backtrace = Backtrace.capture()

print(backtrace)

var symbolicated = backtrace.symbolicated()

print(symbolicated)
```

## Source compatibility

This proposal is entirely additive.  There are no source compatibility
concerns.

## Effect on ABI stability

The addition of this API will not be ABI-breaking, although as with any
new additions to the standard library it will constrain future versions
of Swift to some extent.

## Effect on API resilience

Once added, some changes to this API will be ABI and source-breaking
changes.  Changes to the new structs/classes will be restricted as
described in the [library evolution
document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst)
in the Swift repository.

## Alternatives considered

This could have been addressed by creating a separate Swift package,
or by updating the existing [swift-server/swift-backtrace
package](https://github.com/swift-server/swift-backtrace).

The latter focuses explicitly on Linux and Windows, and has
significant limitations, in addition to which we would like for this
functionality to be built in to Swift---just as it is built into
competing languages.  This is why we felt it should be built into the
Swift runtime itself.

The `Address` type could have been a fixed width integer, but that
loses some flexibility, both in terms of backtrace storage, and in our
ability to cope with backtraces from a platform other than the host.
Using a protocol here means that we can choose an appropriate
underlying implementation, without limiting ourselves in the future;
it also avoids having the type differ on different platforms.

The `frames` member variables could have been arrays, but implementing
them instead as a sequence means that we have the flexibility to use
a different backing store where doing so makes sense.  An example where
we might want that is where we're capturing very large numbers of
backtraces, in which case doing some kind of delta compression on the
frame addresses might enable us to save significant amounts of memory.
Additionally, using a sequence here means we can use a concrete `Address`
type for storage, rather than having to use `any Address`; the latter
only need appear at the interface.

## Acknowledgments

Thanks to Jonathan Grynspan and Mike Ash for their helpful comments
on this proposal.
