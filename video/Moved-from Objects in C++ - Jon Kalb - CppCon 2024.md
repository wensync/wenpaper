
# [Moved-from Objects in C++ - Jon Kalb - CppCon 2024](https://www.youtube.com/watch?v=FUsQPIoYoRM)

summary from https://notegpt.io/youtube-subtitle-downloader

## Introduction to C++ and its superpower

The speaker introduces the talk titled 'This is C++' and sets the expectation that the audience is already familiar with C++. The focus is on understanding what C++ is, what it aims to accomplish, and how it achieves those goals. This understanding is crucial both for effectively using C++ and for guiding its evolution through new language and library features. The talk will cover examples illustrating these points, address a major controversy in the C++ community, and discuss the implications for future challenges. The speaker begins by posing the question of C++'s superpower, noting that while C++ has several strengths, none are entirely unique to the language.

## C++ superpower: uncompromising performance

C++'s core strength lies in its uncompromising performance, prioritizing speed over trade-offs such as safety. The standards committee ensures that while C++ offers high-level abstractions for easier code reasoning, it never sacrifices performance. This is encapsulated in the zero-overhead rule, meaning features that aren't used add no overhead, and features that are used add no more overhead than necessary to implement them manually.

C++ achieves high performance by directly translating user code into machine instructions when possible. For example, simple operations like integer addition and division map directly to single machine instructions. However, safety concerns arise with these operations, such as integer overflow in addition and multiplication, and division by zero errors. The challenge is for C++ compilers to generate efficient, safe code across all platforms, assuming no hardware bugs.

## Handling corner cases and undefined behavior

The discussion focuses on how different computing platforms handle corner cases differently, while non-corner cases yield consistent results across platforms. Defining uniform behavior for corner cases in a language could ensure consistency, but emulating such behavior on platforms that don't natively support it can significantly slow down common, non-corner-case operations. For example, in C++, behavior for corner cases like division by zero is undefined to avoid this overhead. Some languages might choose to define safe behaviors, such as returning the maximum integer value or throwing exceptions. However, implementing even simple safety checks like returning a max value on division by zero can double the number of instructions executed and introduce performance costs due to additional tests and branching.

## Safe vs unsafe operations in C++ standard library

The discussion contrasts safe and unsafe C++ operations using the example of std::vector's element access. The index operator offers fast access without range checking (unsafe), while the 'at' function provides range-checked access (safe). In practice, the index operator is almost always used because it avoids repeated checks, especially in loops where the programmer ensures valid indices. Although shifting range checking from the library to the caller might seem equivalent, performing checks once outside repeated accesses is more efficient. This illustrates the principle that safe operations can be built on top of fast ones, but fast operations cannot be built on top of inherently safe ones.

## Building safe APIs on top of fast APIs

The speaker explains that having an API without built-in checking allows for faster performance by letting external code handle necessary checks. In C++, minimal domain checking is done in library calls to avoid unnecessary overhead, placing the responsibility on the programmer to ensure safety. This design choice leverages C++'s 'undefined behavior' to achieve uncompromising performance, accepting that the programmer must avoid unsafe cases.

The talk continues on C++ performance trade-offs, emphasizing that programmers must carefully express code to avoid corner cases that prevent optimized execution. The example discussed applies to C++ standards up to 23, with some changes expected in C++ 26. The presenter highlights that variables of fundamental types can be declared without initialization, which contradicts naive expectations that such variables default to zero.

Uninitialized variables are common sources of bugs, but initializing every variable by default would hurt performance, especially when variables are immediately assigned meaningful runtime values. The choice in C++ favors performance over safety. The speaker suggests that the language syntax could be safer without performance loss by explicitly marking zero initialization versus no initialization, making intentions clearer.

The speaker proposes a syntax distinction where declaring a variable alone means zero initialization, and a special notation indicates no initialization, clarifying programmer intent. They note that depending on undefined behavior is itself undefined, so changing the standard to zero-initialize uninitialized variables in the future would likely not break existing code. This change could fix many bugs caused by uninitialized variables currently found in the wild.

## Uninitialized variables and performance trade-offs

The discussion focuses on the behavior of uninitialized objects in C++. It is legal to assign to an uninitialized variable and to pass it by reference, but reading its value is undefined behavior according to the standard. Taking the address of such an object is debated but generally not considered undefined behavior. An uninitialized object can also be ignored or destroyed without issues.

The speaker explains that while some compilers and static analyzers can warn about uninitialized variables in simple cases, they generally cannot detect undefined behavior in all scenarios, especially across function calls or translation units. Without whole program analysis, detecting such issues reliably is impossible. The speaker questions whether function preconditions should explicitly state that parameters must be initialized, concluding that it's usually assumed and not documented.

It is emphasized that passing uninitialized objects into functions by non-const reference can cause serious problems. Although uninitialized objects are rarely created or considered in daily programming, they are a defined part of the language. The segment hints at further formal definitions related to uninitialized objects by notable programming authors.

## Partially formed objects and programmer responsibility

The speaker explains the concept of partially formed objects in C++, where the compiler knows an object's static type but its runtime value isn't fully formed. This creates risks for inexperienced programmers. They use a ski slope analogy to illustrate how staying within defined behavior ensures safe, fast code, but venturing near undefined behavior can lead to serious issues. The tradeoff is accepted because C++ prioritizes performance and leaves the burden of avoiding corner cases to the programmer.

The talk continues on why C++ has undefined behavior, emphasizing that it enables uncompromisingly fast performance. Although this approach imposes safety trade-offs and places responsibility on programmers, many consider C++ unsafe. However, this view misses the broader context: applications are built not just by compilers but by the entire C++ ecosystem—including tooling, testing, libraries, training, and programmer expertise—that collectively manage safety concerns.

The C++ ecosystem is highly aware of memory safety and continually evolves standards to improve safety without sacrificing performance. The upcoming C++26 standard incorporates new tools to enhance memory safety, reliability, and performance. The speaker warns against relying solely on compilers for safety, as this misunderstanding can lead to misguided safety measures that reduce performance. Additionally, logic errors exist regardless of language safety features, and an overreliance on new memory-safe languages without adequate ecosystem support or training can lead to expensive, buggy software.

## C++ ecosystem's role in safety and performance

The segment discusses how well-trained engineers using advanced tools can develop C++ applications that outperform and are as safe or safer than those created in other languages with less mature ecosystems. It highlights that the industry's investment in C++ ensures both performance and potential cost advantages. However, mandating safety in the language does not guarantee perfectly safe or efficient applications, only that performance tradeoffs are made consciously. The segment also introduces modern C++ (C++11), which added move semantics and rvalue references, noting that these features complicate the language but are essential concepts for modern C++ developers.

## Move semantics and rvalue references in modern C++

The discussion begins with the complexities of value semantics, rvalue references, and move semantics in C++, highlighting their cognitive challenges even for experienced programmers. Despite their difficulty, these features are essential for eliminating unnecessary copies and maximizing performance. The speaker emphasizes that C++ prioritizes performance, which justifies the added language complexity.

The focus shifts to move semantics in constructors, particularly move-initializing data members to avoid expensive copies. The speaker questions the characteristics of move constructors, noting they should be cheap, constant-time, and non-throwing. Using the vector move constructor as an example, it is confirmed that it meets these criteria, being cheap and noexcept, although the standard only guarantees constant time for move constructors.

A critique of the standard committee's decision is presented, suggesting that the guarantee of constant time for move constructors is a compromise that undermines performance. The speaker proposes a review of move semantics, excluding legacy code or types without defined move operations, focusing instead on types with well-defined, movable data members.

To ensure cheap, constant-time, non-throwing move operations, the speaker outlines a classification of data members into three categories: primitive types, owning pointers, and compound types composed of these. By following simple rules such as shallow copying primitives and zeroing out the source, all types can be designed to have efficient move semantics.

## Implementing efficient move operations

The segment discusses the handling of move operations for compound types, emphasizing that destructors must safely operate on moved-from objects without making invalid assumptions. It highlights the importance of supporting safe assignment to moved-from objects, especially in algorithms involving sorting, inserting, removing, and unique operations on random access containers.

This part explains the early proposal for move semantics (N1377), which required moved-from objects to be left in a self-consistent state preserving all internal invariants, primarily to allow safe destruction and assignment. However, many users misinterpreted this, treating moved-from objects like partially formed objects with known static types but not fully valid runtime values. The standard, however, defines moved-from standard library objects as valid states that support all operations without preconditions, effectively considering them fully formed, which has important implications for container types like vector and list.

## Move semantics implications for standard containers

The standard does not specify how Vector is implemented, but it typically contains three data members: a pointer to the allocated array, a size tracker, and a capacity tracker. These can be implemented using integers or pointers. For example, an allocated integer array might have a capacity of 16 but currently hold 10 elements. When moving from such a vector, the allocation pointer is set to nil while primitive types like size and capacity are copied unchanged, representing a minimal move operation.

A challenge arises when calling size() on a moved-from vector because size must always be valid according to the standard. If the size member function simply returns the size value, it may no longer be accurate after a move. One solution could be to check if the vector is moved-from before returning size, but this adds a performance cost for every size call. Therefore, the standard implementation must zero out size and capacity during a move to maintain correctness, which introduces a small but unavoidable overhead.

Although zeroing out three words (size, capacity, pointer) during a vector move may be a minor performance cost, the situation is more complex for the list container. The list standard does not require move constructors to be noexcept. This is because some list implementations use a sentinel node — a special node marking the end of the list that holds no data but helps with traversal termination.

The sentinel node in lists must be allocated and deallocated along with the list nodes, which is straightforward during construction and destruction. However, during a move construction, the new list initially has no sentinel. It makes sense to transfer the sentinel from the moved-from list to the new one. After this, the moved-from list must still maintain a sentinel if it is expected to behave normally, requiring allocation of a new sentinel, which can potentially throw exceptions. This explains why list move constructors cannot guarantee noexcept, highlighting a trade-off in the design and implementation of standard containers.

## Move constructor challenges with sentinel nodes

The discussion explains why library implementers chose to use Sentinels despite implications for move operations. This decision predates the concept of moves and was intended to avoid breaking existing binary code. The standards committee acknowledged the trade-offs and decided not to enforce noexcept on move constructors to maintain compatibility. While the additional cost to vector is minor, it can be significant for node-based types like lists. The speaker criticizes the committee's guidance on the state of moved-from objects, arguing that checking such state encourages logic errors. Instead, once an object is moved from, its state should not be relied upon; if the state matters, the object should not be moved from in the first place.

## Recommended usage of moved-from objects

The speaker explains that when using move semantics in programming, a moved-from object should only be either destroyed or assigned a new value. This responsibility lies with the programmer, not the compiler. Although this may seem like a burden, it is manageable because one only moves from an object when they intend to assign or destroy it.

The speaker shares a conversation with a standard library implementor named Anigo Montoya about the lack of other use cases for moved-from objects beyond destruction or assignment. Anigo offers an example where a vector is moved from and then reused, assuming it is empty. However, the standard does not guarantee that a moved-from object is empty, and relying on this assumption can lead to performance costs, making it less efficient than simply assigning a new value.

The speaker discusses why the standards committee enforces the rule that moved-from objects must only be assigned or destroyed despite performance impacts. The committee avoids allowing moved-from objects to have unstable invariants because library implementors find it uncomfortable to manage objects in states where invariants don’t hold. The speaker compares moved-from objects to objects whose lifetime is suspended—their state is frozen, and no operations besides assignment or destruction should be performed.

The speaker expands on the concept that a moved-from object’s lifetime is suspended, awaiting destruction or reassignment, and no assumptions should be made about its state or invariants. They reference a 2020 blog post by Herb Sutter that explains well-behaved moved-from objects do not require special restrictions, reinforcing the idea that improper use of moved-from objects should be avoided.

## Different views on moved-from object handling

The speaker discusses a blog post aimed at easing programmers' concerns about moving from objects, referencing Herb's view that move operations should be treated like any other object operation without extra mental overhead. The speaker respectfully disagrees with Herb, who is both their boss and a respected community leader. They highlight a difference of technical opinion and introduce the term 'well-behaved types' used in the blog post.

The speaker explains that while the standards committee cannot dictate how code is written, it effectively imposes requirements on objects used in standard library components. For example, objects must meet certain criteria to be usable in containers or threading. The speaker encourages reading Herb's blog and especially the comments by Shawn Parent, who notes that certain comments in the standard are explanatory (non-normative) but imply that requirements on objects remain even after they are moved from.

The discussion continues with Shawn Parent's observation that invariants on objects must hold even after they are moved from, illustrated through a thought experiment involving an array of unique pointers. This emphasizes that certain conditions must still be met by objects passed into library functions despite being in a moved-from state.

## Unique pointer sorting and move-from object issues

The segment discusses sorting a pointer array where each pointer points to an object. The sorting uses a comparator that dereferences unique pointers to compare their values rather than their addresses. While a properly implemented sort function should handle this correctly, it violates a note in the standard because after a unique pointer is moved from, it cannot be dereferenced. However, the sort never compares moved-from objects, ensuring it works despite this undefined behavior.

The explanation continues by clarifying that the standard's note leads to adding performance-costly checks to avoid dereferencing empty unique pointers, though these checks are often unnecessary. This issue was raised by Shan, who suggests the note should specify that moved-from objects are assignable but not otherwise specified. The speaker references Shan's talk 'Find is Broken' for further details and critiques the blog's approach as creating an illusion of safety by allowing calls on moved-from objects that may be logically invalid.

The speaker reflects on their research, which led them to consult C++ experts Howard Hinn and Shan Parent. Howard was a pioneer in move semantics, and Shan highlighted the issue with Herb Sutter's blog post on the topic. Shan's perspective influenced the talk, emphasizing the nuanced problems with moved-from states and the standard's handling of them, contrasting with Howard's approach.

## Debate on library requirements for moved-from states

The discussion centers on how to handle move-from objects in programming standards. One perspective advocates for explicitly defining requirements for every algorithm, emphasizing that move-from objects should only be given new values or destroyed. The contrasting view supports allowing operations like sorting containers with move-from objects, provided the comparator can handle them, although this is considered a logic error by others who believe such use should be avoided.

The conversation shifts to interpreting function signatures related to object state after modification. One interpretation requires that the function must leave the object in a valid state without violating invariants. Another suggests the object may be left in a partially formed state. The debate questions whether these signatures should be considered equivalent and explores implications using a hypothetical example involving a complex 3D rendering object driven by AI, highlighting the importance of clearly defining object state post-modification.

## Example: wrapping OS viewport with move semantics

The speaker explains the concept of creating and freeing a viewport using viewport IDs, which act as pointers or indices to data structures in the operating system's memory. They highlight operations that can be performed with viewport IDs, such as retrieving dimensions, specifically the X dimension.

The speaker proposes wrapping the viewport functionality in a C++ class. The class constructor calls create viewport, storing the viewport ID as a single data member. The destructor frees the viewport. Accessor methods provide viewport details. This approach offers a clean object-oriented mapping to the underlying OS functionality.

The discussion moves to implementing move semantics for the viewport class. The move operation transfers the viewport ID from the source object to the destination and zeros out the source's ID. This ensures safe destruction, as the destructor only frees the viewport if the ID is valid.

The speaker outlines assignment operations: move assignment shifts the viewport ID between objects, while copy assignment would require creating a new viewport, which is expensive. This design manages resource ownership and lifecycle carefully to avoid leaks or redundant operations.

A problem arises when calling methods like getting the X dimension after a move, as the new object may lack a valid viewport ID to pass to OS functions. One proposed but costly solution is to create a new viewport on each move. The speaker critiques this approach due to its high expense and hints at exploring better alternatives.

## Challenges with moved-from object member functions

The speaker discusses the challenges of handling moved-from objects in C++. One proposed solution is to modify member functions to return fake or default values when called on moved-from objects, but this approach requires extensive code checks and adds complexity and performance overhead. The speaker rejects this solution and suggests that moved-from objects should only be destroyed or assigned new values, emphasizing that C++ involves uncompromising behavior and that this approach aligns with the language's design philosophy.

## Summary: C++ trade-offs and undefined behavior

The speaker discusses the trade-offs involved with undefined behavior in C++ and the standards committee's stance on it. While undefined behavior allows compilers to generate fast performance if programmers stay within defined bounds, the standards committee has only noted this issue without fully resolving it. The committee has decided that objects moved from are still considered fully formed types, a decision the speaker regards as a mistake due to causing both performance hits and logic errors. Changing this behavior now is problematic because existing code depends on it.

The committee aims to reduce the cognitive load on programmers by treating moved-from objects as fully formed, eliminating the need for programmers to worry about their state after a move. However, the speaker argues this leads to either poor performance or complex, hard-to-maintain code. Instead, the speaker advocates that moved-from objects should be considered partially formed, with only two valid actions allowed: assigning a new value or letting the object go out of scope. This approach aligns better with safety considerations in C++ programming.

## Future of C++ safety and performance balance

The discussion begins with the presenter emphasizing that C++ is evolving to be safer without sacrificing performance. A question arises regarding unique pointers and their preconditions, specifically about calling functions on moved-from objects. The presenter explains that although the standard states comparators must work with moved-from objects, in practice, code like sorting should never compare moved-from objects, and the comparator design assumes that. The code written by Shawn avoids dereferencing null unique pointers during swaps, which aligns with proper usage.

The conversation continues on the topic of comparators and move semantics, highlighting that the standard requires comparators to work even with moved-from objects, though in practice this scenario should not occur. The challenge of self-move assignment is brought up, especially in generic or templated code, where the burden on the caller to detect self-move can be problematic. The presenter notes that Howard Hinnant has shifted his position toward building safety into move operations to handle such cases, though the presenter does not take a firm stance on this debate.

A question is posed about the C++ community’s priorities in light of security professionals and governments recommending against C++, drawing a parallel to automotive safety standards. The presenter argues that the goal should be to make C++ as safe as possible without compromising its hallmark performance. Unlike languages that prioritize safety at the expense of speed, C++ aims to maintain high performance while improving safety. The presenter cites examples such as fighter planes running on C++ to show that safe, high-performance C++ is achievable with the right tools and education.

The conversation shifts to a practical scenario involving unit tests for movable types that own resources like sockets. The question is whether static analysis warnings about operations on moved-from objects should be ignored or if tests should be redesigned to avoid such warnings. The presenter acknowledges the complexity of this issue, admitting to not having a ready solution and indicating the need for further consideration before making decisions during code reviews.

An inquiry is made about the existence or possibility of a 'move sanitizer' tool analogous to thread sanitizers. The presenter is uncertain if such a tool exists and suggests that the standards committee is moving toward a particular approach on move semantics, though not unanimously. The presenter highlights that the direction seems to be building safety features around move operations, but acknowledges that consensus is not fully settled.

The presenter reflects on the evolution of move semantics understanding since 2020. Initially, the focus was on performance, with the standard committee and library implementers aiming to keep moved-from objects in a valid but unspecified state, typically reset to empty. Exceptions like node-based containers with sentinels complicate this model but overall the approach is seen as a reasonable overhead tradeoff for safety. The presenter notes a divergence between teaching users and focusing on library implementation, emphasizing the need for better education on handling moved-from objects.

The presenter concludes by discussing Herb Sutter's stance that if the standard library can handle moved-from objects safely, then user code should be able to as well, reducing the cognitive burden on developers. However, the presenter warns that there are complexities not fully appreciated by the standards committee. The session runs slightly over time, and the presenter thanks the audience and prepares to close the discussion.
