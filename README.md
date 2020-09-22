# 6771-20T2

# Assignment 1 - Word Ladder

**Please `git pull` frequently to get the latest changes.**

## Change Log

* `vcpkg` triplets renamed for home use + PowerShell script updated for PowerShell compatibility.
* "awake" -> "sleep" example updated with missing word
* (11th June) "awake" -> "sleep" example updated properly
* (13th June) Assumption that start word != end word made clear
* (13th June) Clarity that you don't _have_ to put all the tests in one file
* (15th June) Adds gsl-lite to root CMakeLists.txt

## Overview

In Week 2 we are learning about C++ libraries, and this assignment is your chance to practice those
skills.

Now that you’ve been introduced to the C++ standard library, it’s time to put that knowledge to use.
In the role of client, the low-level details have already been dealt with, and you can focus your
attention on solving more pressing problems. Having a library of thoroughly tested and reviewed
types designed by field experts vastly broadens the kinds of tasks you can “easily” tackle. In this
first assignment, you’ll write the back-end for a program that heavily leverages the standard
library to do nifty things. It might sound a little daunting at first; but given the power tools in
your arsenal, you can focus on solving the problem at hand, instead of worrying about how to
implement everything from scratch. Let’s hear it for abstraction!

This assignment has several purposes:

1. To explore C++’s value-semantics.
2. To stress the notion of abstraction as a mechanism for managing data and providing functionality
   without revealing the implementation.
3. To become more familiar with C++'s type system.
4. To gain familiarity with the C++ standard library and several widely used third-party libraries.

Leveraging `std::vector`, `std::queue`, and `absl::flat_hash_set` are critical for writing a
word ladder builder. A word ladder is a connection from one word to another, formed by changing one
letter at a time, with the constraint that each transformation yields a new valid word. For example,
here is a word ladder connecting "code" to "data".


```txt
code -> cade -> cate -> date -> data
```

Many word ladders will have mutliple solutions, such as connecting "work" to "play".

```txt
work fork form foam flam flay play
work pork perk peak pean plan play
work pork perk peak peat plat play
work pork perk pert peat plat play
work pork porn pirn pian plan play
work pork port pert peat plat play
work word wood pood plod ploy play
work worm form foam flam flay play
work worn porn pirn pian plan play
work wort bort boat blat plat play
work wort port pert peat plat play
work wort wert pert peat plat play
```

The back-end that you will write takes a start word, a destination word, and a lexicon, which
returns valid word ladders. By using a [breadth-first search][bfs], you’re guaranteed to find the
shortest such sequence. Here are some more examples.

```cpp
::word_ladder::generate("work", "play", english_words);
// returns {{"work", "fork", "form", "foam", "flam", "flay", "play"}}
::word_ladder::generate("awake", "sleep", english_words);
// returns {
//   {"awake","aware","sware","share","sharn","shawn","shewn","sheen","sheep","sleep"}
//   {"awake","aware","sware","share","shire","shirr","shier","sheer","sheep","sleep"}
// }
::word_ladder::generate("airplane", "tricycle", engish_words);
// returns {}
```

## Understanding a word ladder implementation

Finding a word ladder is a specific instance of a [shortest-path problem][ssp], where the challenge
is to find the shortest path from a starting position to a goal. Shortest-path problems come up in a
variety of situations, such as packet routing, robot motion planning, social networks, and studying
gene mutations. One approach for finding a shortest path is the classic breadth-first search
algorithm. A breadth-first search searches outward from the start in a radial fashion, until it hits
the goal. For our word ladder, this means examining those ladders that represent one hop from the
start. A “hop” is a change in letter. One “hop” from the start means one changed letter, two “hops”
means two changes in letters, and so on. It’s possible for the same position to change letters
across multiple non-adjacent hops. If any of these reach the destination, we’re done. If not, the
search now examines all the ladders that add one more hop. By expanding the search at each step, all
one-hop ladders are examined before two-hop ladders, and three-hop ladders are only taken into
consideration if none of the one-hop or two-hop ladders work out; thus the algorithm is guaranteed
to find the shortest successful ladder.

Breadth-first searches are typically implemented using a queue. The queue stores partial ladders
that represent possibilities to explore. The ladders are enqueued in order of increasing length. The
first elements enqueued are all the one-hop ladders, followed by the two-hop ladders, and so on. Due
to FIFO handling, ladders will be dequeued in order of increasing length. The algorithm operates by
dequeuing the front ladder from the queue and determining if it reaches the goal. If it does, then
you have a complete ladder, and it is the shortest. If not, you take that partial ladder and extend
it to reach words that are one more hop away, and enqueue those ladders onto the queue, to be
examined later. If you exhaust the queue of possibilities without having found a completed ladder,
you can conclude that no ladder exists.

A few of these tasks deserve a bit more explanation. For example, you’ll need to find all the words
that differ by one letter from a given word. You might reach for a raw loop to change each letter to
all the other letters in the alphabet; or you can explore the libraries that the project uses to
solve the problem in a more descriptive manner (note 2). Repeat this for each letter position in the
word and you will have discovered all the words that are one letter away.

Another, more subtle issue, is the restriction that you shouldn’t reuse words that have been
included in a previous ladder. This is an optimisation that avoids exploring redundant paths. For
example, if you previously tried the ladder `cat->cot->cog` and are now processing `cat->cot->con`,
you would find that the word `cog` is one letter away from `con`, which looks like a potential
candidate to extend this ladder. However, `cog` has already been reached in an earlier (and thus
shorter) ladder, so there is no point in reconsidering it in a longer ladder. The simplest way to
ensure this is to keep track of the words that have been used in any ladder, and ignore them when
they resurface. This is also necessary to avoid getting trapped in circular, non-terminal ladders
such as `cat->cot->cog->bog->bat->cat`. Since you need linear access to all of the items in a
word ladder when time comes to return it, it makes sense to model an individual word ladder using
`std::vector<std::string>`. Remember that because C++ has value semantics, you’re able to copy
vectors via copy initialisation (e.g. `auto word_ladder_clone = word_ladder;`) and copy assignment
(e.g. `word_ladder_clone = word_ladder`).

**If there are multiple shortest paths, your implementation must return all the solutions, sorted in
lexicographical order. Thus, the return type for your word ladder generator will be
`std::vector<std::vector<std::string>>`.**


## The Task

This generator might seem like it’s a panacea, but it still benefits from a step-by-step development
plan to keep things moving along.

* **Task 1** --- *Familiarise yourself with the libraries available.* You don’t need to deep-dive,
but it would be a smart idea to read up on `std::vector`, `std::queue`, and `std::unordered_set`,
from the standard library; and to read up on the range adaptors we’ve listed in the appendix. You
shouldn’t worry about their implementation details: focus on the interfaces, and how you use them in
practice.
* **Task 2** --- *Test design.* We’ve provided you with a very simple test case to show you how to
use the test library. You should add more `TEST_CASE`s underneath, so you have a suite of checks to
make sure you catch any logic errors in your generator. We adopt Google’s Beyoncé rule in this
class: “if you liked it, you should have put a test on it”. Test words that are short (one or two
letters) and test words that are longer.
* **Task 3** --- *Design your algorithm.* Be sure you understand the breadth-first search algorithm
on paper, and what types you will need to use.
* **Task 4** --- *Lexicon handling.* Set up an `absl::flat_hash_set` object with the large lexicon,
read from our data file. There’s a utility function called `word_ladder::read_lexicon` that will
read it in from file for you.

### Assumptions

* You can assume that the start word and end word have the same length (i.e. number of characters).
* You can assume that both the start and the end word are in the lexicon.
* You can assume the start word and the end word will not be the same word

### Implementation hints

Again, it’s all about leveraging the libraries at your disposal --- you’ll find your job is just to
coordinate the activities of your objects to do the search.

* [`std::vector`][vector] maintains a contiguous sequence of elements, and has random access.
  Benchmarks have shown that its contiguous storage makes it the fastest and smallest container
  in many situations (even when computer science theory tells us that a linked list would be
  better!).
* [`std::queue`][queue] offers a FIFO interface over a container, which is probably what you’ll
  want to use for tracking your partial ladders. Ladders are enqueued (and thus dequeued) in
  order of length, so as to find the shortest option first.
* [`absl::flat_hash_set`][fhs] (note 1) is the hash-set that we use for the input lexicon, and is
  recommended for any internal lexicons you might use.
* [Algorithms] and ranges are powerful tools for describing what you want your program to do.
* As a minor detail, it doesn’t matter if the start and destination words are contained in the
  lexicon or not. You can eliminate non-words from the get-go or just allow them to fall through and
  be searched anyway. During marking, the start and destination words will always be taken from the
  lexicon.

## Getting Started

If you haven't done so already, clone this repository.

```sh
$ git clone gitlab@gitlab.cse.unsw.edu.au:z5555555/20T2-cs6771-ass1
```

(Note: Replace z5555555 with your zid)

Navigate inside the directory. You can then open vscode with `code .` (not the dot).

Refer to `include/comp6771/word_ladder.hpp` for help starting your `source/word_ladder.cpp` file.

### Running the tests

Similar to the first tutorial, you simply to have to run `Ctrl+Shift+P` and then type `Run Test` and
hit enter. VS Code will compile and run all of your tests and produce an output.

### Adding more tests

Part of your assignment mark will come from the quality and extensiveness of tests that you write.

You can add more test files to the `test/word_ladder/` directory. Simply copy `test/word_ladder/word_ladder_test1.cpp`
into another file in that directory.

Note, everytime that you add a new file to the `test/word_ladder/` directory you will need to add
another few lines to `test/CMakeLists.txt`. You can once again, simply copy the test reference for
`word_ladder_test1.cpp` and rename the appropriate parts. Every time you update `CMakeLists.txt` in
any repository, in VSCode you should press `Ctrl+Shift+P` and run `Reload Window` for the changes to
take effect.

### Measuring your performance

While you aren't assessed on the performance of your code, there is a **15 second time limit** per
test that we will enforce when marking your assignment. For the average student, this means that
some of the hardest test cases may not run fast enough with your algorithm and you may fail the latter
tests due to timeout.

We have provided the hardest test case we will test against in `test/word_ladder/word_ladder_test_benchmark.cpp`.
If you can successfully pass this test within 15 seconds **ON THE CSE MACHINE** then we are providing
you a guarantee that you will not fail those tests from a timeout. If you exceed 15 seconds for that
test, you may still get the marks, but it's just something we can't guarantee.

Please note: This benchmark test is VERY difficult. If your code takes an hour to run this test that's
not something that should make you panic. It's quite easy to fall in the time limit for most of the
tests. Don't stress and just make sure earlier tests fall in the 15 second time limit on CSE
machines.

To measure your performance:

1. In VSCode, down the very bottom of the window, change your Cmake from `[Debug]` to `[Release]`.
   This will remove debug symbols and other things, which will mean your code runs faster, but is
   near impossible to debug for mere mortals. This is usually what is done when you're finished
   developing and ready for release. We will discuss this more in depth later.

![Debug to release picture](debug.png)

2. In VSCode, press `Ctrl+Shift+P` and select `Build Taret`. Once the next textbox comes up, type
   `benchmark` and select that test to build

3. In your project directory (the one that has the README.md file in it) run `bash benchmark`. It
   will output something like this below. The time you should measure yourself against is the "user"
   time. If this is under 15 seconds then you're all good.

```txt
real    1m45.089s
user    1m44.497s
sys     0m0.560s
```

4. In VSCode, down the very bottom of the window, change your Cmake from `[Release]` to `[Debug]`.
   Now that you're done doing a sanity check benchmark, leave debug symbols on so that you can more
   effectively debug your code.

## Marking Criteria

This assignment will contribute 15% to your final mark.

The assessment for the assignment recognizes the difficulty of the task, the importance of style,
and the importance of appropriate use of programming methods (e.g. using while loops instead of a
dozen if statements).

<table class="table table-bordered table-striped">
  <tr>
    <td align=right>50%</td>
    <td>
      <b>Correctness</b><br />
      The correctness of your program will be determined automatically by tests that we will run against
      your program. You will not know the full sample of tests used prior to marking. Your program must
      also find the word ladder(s) for a given input in the time limit specified above.
    </td>
  </tr>
  <tr>
    <td align=right>25%</td>
    <td>
      <b>Your tests</b><br />
      You are required to write your own tests to ensure your program works.
      You will write tests in the <code>test/</code> directory. At the top of each file you will also include a block comment to explain the rational and approach you took to writing tests. Please read the <a href="https://github.com/catchorg/Catch2/blob/master/docs/tutorial.md">Catch2 tutorial</a> or review lecture/tutorial content to see how to write tests. Tests will be marked on several
      factors. These include, but are not limited to:
      <ul>
        <li>Correctness — an incorrect test is worse than useless.</li>
        <li>
          Coverage - your tests might be great, but if they don't cover the part that ends up
          failing, they weren't much good to you.
        </li>
        <li>
          Brittleness — If you change your implementation, will the tests need to be changed (this
          generally means avoiding calling functions specific to your implementation where possible
          - ones that would be private if you were doing OOP).
        </li>
        <li>
          Clarity — If your test case failed, it should be immediately obvious what went wrong (this
          means splitting it up into appropriately sized sub-tests, amongst other things).
        </li>
      </ul>
    </td>
  </tr>
  <tr>
    <td align=right>20%</td>
    <td>
      <b>C++ best practices</b><br />
      Your adherence to good C++ best practice in lecture. This is <b>not</b> saying that if you
      conform to the style guide you will receive full marks for this section. This 20% is also
      based on how well you use modern C++ methodologies taught in this course as opposed to using
      backwards-compatible C methods. Examples include: Not using primitive arrays and not using
      pointers. We will also penalise you for standard poor practices in programming, such as having
      too many nested loops, poor variable naming, etc.
    </td>
  </tr>
  <tr>
    <td align=right>5%<td>
    <b>clang-format</b><br />
    In your project folder, run the following commands on all cpp/h files in the `source` and `test` directory.<br />
    <code>$ clang-format-11 -style=file -i /path/to/file.cpp</code>
    If, for each of these files, the program outputs nothing (i.e. is linted correctly), you will receive full marks for
    this section (5%).
    A video explaining how to use clang-format can be found [HERE][hayden-tutorial]
  </tr>
</table>

The following actions will result in a 0/100 mark for Word Ladder, and in some cases a 0 for
COMP6771:

* Knowingly providing your work to anyone and it is subsequently submitted (by anyone).
* Submitting any other person's work. This includes joint work.

The lecturer may vary the assessment scheme after inspecting
the assignment submissions but it will remain broadly similar to the description above.

## Originality of Work

The work you submit must be your own work.  Submission of work partially or completely derived from
any other person or jointly written with any other person is not permitted.

The penalties for such an offence may include negative marks, automatic failure of the course and
possibly other academic discipline. Assignment submissions will be examined both automatically and
manually for such submissions.

Relevant scholarship authorities will be informed if students holding scholarships are involved in
an incident of plagiarism or other misconduct.

Do not provide or show your assignment work to any other person &mdash; apart from the teaching
staff of COMP6771.

If you knowingly provide or show your assignment work to another person for any reason, and work
derived from it is submitted, you may be penalized, even if the work was submitted without your
knowledge or consent.  This may apply even if your work is submitted by a third party unknown to
you.

Note you will not be penalized if your work has the potential to be taken without your consent or
knowledge.

## Submission

This assignment is due *Friday 19th of June, 19:59:59*.
Submit the assignment using the following comand while logged into the CSE machines:

```sh
6771 submit ass1
```

This will submit whatever is on the master branch of THIS repository (the one this README.md file is
contained in) at the moment of submission.

Please ensure that you can build and run your tests successfully on the CSE machine.

## Late Submission Policy

If your assignment is submitted after this date, each hour it is late reduces the maximum mark it can achieve by 0.5%.

For example if an assignment you submitted with a raw awarded mark of 85% was submitted 5 hours late, the late submission would have no effect (as maximum mark would be 97.5%).

If the same assignment was submitted 40 hours late it would be awarded
80%, the maximum mark it can achieve at that time.

This late penalty has been amended from the original specification, and you should not assume it will be the same for future assignments.

## Appendix

* (note 1) We link you to the C++ Reference page for `std::unordered_set` because
  `absl::flat_hash_set` is almost interface-compatible; the Abseil documentation links to C++
  Reference’s `std::unordered_set` docs anyway, and mostly explains the advantages of
  `absl::flat_hash_set`.
* (note 2) You’ll need to consult the lexicon to see if the transformation is a valid word in either
  case.

[fhs]: https://en.cppreference.com/w/cpp/container/unordered_set
[Algorithms]: https://en.cppreference.com/w/cpp/algorithm
[bfs]: https://en.wikipedia.org/wiki/Breadth-first_search
[ssp]: https://en.wikipedia.org/wiki/Shortest_path_problem
[queue]: https://en.cppreference.com/w/cpp/container/queue
[vector]: https://en.cppreference.com/w/cpp/container/vector
[hayden-tutorial]: https://unsw.zoom.us/rec/share/4_1tE-mo2UpJRo2V0U_UVp4BRoDAeaa8hicbrvVfxRs2wxZeMb3Au5H-75jpZZWj


# Assignment 2 - Euclidean Vector

**Please `git pull` frequently to get the latest changes.**

## Change Log

* 26/06: Fixing up test stub
* 26/06: Moved euclidean norm comment to appropriate section, fixed up test case example in README.md
* 29/06: Exception X and Y substitition clarified in throwing exceptions section
* 30/06: Added second exception for `comp6771::unit`
* 05/07: Added exception string clarification for unit vector when euclidean norm is zero - check out exception string
* 07/07: Git submission instructions updated to be VERY clear to push your code to master branch before submitting
* 07/07: Incorrect euclidean_norm pattern used

## The Task

Write a Euclidean Vector Class Library in C++, with its interface given in `include/euclidean_vector.h`
and its implementation in `source/euclidean_vector.cpp`.

We have outlined all key parts of this class below that should be implemented.

### 1. Constructors

**You may have to scroll horizontally to view these tables**

<table border="1" cellspacing="0" cellpadding="5">
  <thead>
    <th>Name</th>
    <th>Constructor</th>
    <th>Description and Hints</th>
    <th>Examples</th>
    <th>Exception: Why thrown & what message</th>
  </thead>
  <tr>
    <td>Default Constructor</td>
    <td><code>euclidean_vector()</code></td>
    <td>
      A constructor that generates a euclidean vector with a dimension of 1 and magnitude of 0.0. <br />
      You can assume the integer input will always be non-negative.
    </td>
    <td>
      <pre><code>(1) auto a = comp6771::euclidean_vector();</code></pre>
    </td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Single-argument Constructor</td>
    <td><code>explicit euclidean_vector(int)</code></td>
    <td>
      A constructor that takes the number of dimensions (as a int) but no magnitudes, sets the magnitude in each dimension as 0.0. <br>
      You can assume the integer input will always be non-negative.
    </td>
    <td>
      <pre><code>(1) auto a = comp6771::euclidean_vector(1);
(2) int i {3};
    auto b = comp6771::euclidean_vector(i);</code></pre>
    </td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Constructor</td>
    <td><code>euclidean_vector(int, double);</code></td>
    <td>
      A constructor that takes the number of dimensions (as an <code>int</code>) and initialises the
      magnitude in each dimension as the second argument (a <code>double</code>). You can assume the
      integer input will always be non-negative.
    </td>
    <td>
      <pre><code>(1) auto a = comp6771::euclidean_vector(2, 4.0);
(2) auto x = int{3};
    auto y = double{3.24};
    auto b = comp6771::euclidean_vector(x, y);</code></pre>
    </td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Constructor</td>
    <td><code>euclidean_vector(std::vector&lt;double&gt;::const_iterator, std::vector&lt;double&gt;::const_iterator)</code></td>
    <td>
      A constructor (or constructors) that takes the start and end of an iterator to a
      <code>std:vector&lt;double&gt;</code> and works out the required dimensions, and sets the
      magnitude in each dimension according to the iterated values.
    </td>
    <td>
      <pre><code>std::vector&lt;double&gt; v;
auto b = comp6771::euclidean_vector(v.begin(),v.end());</code></pre>
    </td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Constructor</td>
    <td><code>euclidean_vector(std::initializer_list&lt;double&gt;)</code></td>
    <td>
      A constructor that takes an initialiser list of <code>double</code>s to populate vector
      magnitudes. You will have to do your own research to implement this one.
    </td>
    <td>
      <pre><code>auto b = comp6771::euclidean_vector{1.0, 2.0, 3.0};</code></pre>
    </td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Copy Constructor</td>
    <td><code>euclidean_vector(euclidean_vector const&amp;)</code></td>
    <td></td>t
    <td><pre><code>auto a = comp6771::euclidean_vector(a);</code></pre></td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Move Constructor</td>
    <td><code>euclidean_vector(euclidean_vector&&)</code></td>
    <td></td>t
    <td><pre><code>auto aMove = comp6771::euclidean_vector(std::move(a));</code></pre></td>
    <td>N/A</td>
  </tr>
</table>

#### Example Usage

```
auto a = comp6771::euclidean_vector(1);      // a Euclidean Vector in 1 dimension, with default magnitude 0.0.
auto b = comp6771::euclidean_vector(2, 4.0); // a Euclidean Vector in 2 dimensions with magnitude 4.0 in both dimensions

auto v = std::vector<double>{5.0, 6.5, 7.0};
auto c = comp6771::euclidean_vector(l.begin(), l.end()); // a Euclidean Vector in 3 dimensions constructed from a vector of magnitudes
```

#### Notes

* You may assume that all arguments supplied by the user are valid. No error checking on constructors is required.
* It's **very important** your constructors work. If we can't validly construct your objects, we can't test any of your other functions.

### 2. Destructor

You must explicitly declare the destructor as default.

For more info look <a href="https://www.ibm.com/developerworks/community/blogs/5894415f-be62-4bc0-81c5-3956e82276f3/entry/defaulted_functions_in_c_11?lang=en">here</a></p>

### 3. Operations

<table border="1" cellspacing="0" cellpadding="5">
  <thead>
    <th>Name</th>
    <th>Operator</th>
    <th>Description</th>
    <th>Examples</th>
    <th>Exception: Why thrown & what message</th>
  </thead>
  <tr>
    <td>Copy Assignment</td>
    <td><code>euclidean_vector& operator=(euclidean_vector const&)</code></td>
    <td>A copy assignment operator overload</td>
    <td><pre><code>a = b;</code></pre></td>
    <td>
      N/A
    </td>
  </tr>
  <tr>
    <td>Move Assignment</td>
    <td><code>euclidean_vector& operator=(euclidean_vector&&)</code></td>
    <td>A move assignment operator</td>
    <td><pre><code>a = std::move(b);</code></pre></td>
    <td>
      N/A
    </td>
  </tr>
  <tr>
    <td>Subscript</td>
    <td>
      <code>operator[]</code>
      <hr />
      A const and non-const declaration is needed
    </td>
    <td>
      Allows to get and set the value in a given dimension of the Euclidean vector. Hint: you may
      need two overloadeds to achieve this requirement.<br />
      <b>Note: It's a requirement you use asserts to ensure the index passed is valid.</b>
    </td>
    <td>
      <pre><code>double a {b[1]};
b[2] = 3.0;</code></pre>
    </td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Unary plus</td>
    <td><code>euclidean_vector operator+()</code></td>
    <td>Returns a copy of the current object.</td>
    <td><pre><code>+a</code></pre></td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Negation</td>
    <td><code>euclidean_vector operator-()</code></td>
    <td>Returns a copy of the current object, where each scalar value has its sign negated.</td>
    <td>
      <pre><code>auto const actual = comp6771::euclidean_vector{-6, 1};
auto const expected = comp6771::euclidean_vector{6, -1};
CHECK(expected == -actual);</code></pre>
    </td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Compound Addition</td>
    <td><code>euclidean_vector& operator+=(euclidean_vector const&)</code></td>
    <td>For adding vectors of the same dimension.</td>
    <td><pre><code>a += b;</code></pre></td>
    <td>
      <b>Given</b>: <code>X = a.dimensions(), Y = b.dimensions()</code>
      <b>When</b>: <code>X != Y</code><br />
      <b>Throw</b>: "Dimensions of LHS(X) and RHS(Y) do not match"
    </td>
  </tr>
  <tr>
    <td>Compound Subtraction</td>
    <td><code>euclidean_vector& operator-=(euclidean_vector const&)</code>
  </td>
  <td>For subtracting vectors of the same dimension.</td>
  <td><pre><code>a -= b;</code></pre></td>
  <td>
    <b>Given</b>: <code>X = a.dimensions(), Y = b.dimensions()</code>
    <b>When</b>: <code>X != Y</code><br />
    <b>Throw</b>: "Dimensions of LHS(X) and RHS(Y) do not match"
  </td>
  </tr>
  <tr>
    <td>Compound Multiplication</td>
    <td><code>euclidean_vector& operator*=(double)</code></td>
    <td>For scalar multiplication, e.g. <code>[1 2] * 3 = [3 6]</code></td>
    <td><pre><code>a *= 3;</code></pre></td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Compound Division</td>
    <td><code>euclidean_vector& operator/=(double)</code></td>
    <td>For scalar division, e.g. <code>[3 6] / 2 = [1.5 3]</code></td>
    <td><pre><code>a /= 4;</code></pre></td>
    <td>
      <b>When</b>: <code>b == 0</code><br />
      <b>Throw</b>: "Invalid vector division by 0"
    </td>
  </tr>
  <tr>
    <td>Vector Type Conversion<br /></td>
    <td><code>explicit operator std::vector&lt;double&gt;()</code></td>
    <td>Operators for type casting to a <code>std::vector</code></td>
    <td>
      <pre><code>auto const a = comp6771::euclidean_vector{0.0, 1.0, 2.0};
auto const vf = static_cast&lt;std::vector&lt;double&gt;&gt;(a);
</code></pre>
    </td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>List Type Conversion<br /></td>
    <td><code>explicit operator std::list&lt;double&gt;()</code></td>
    <td>Operators for type casting to a <code>std::list</code></td>
    <td><pre><code>auto const a = comp6771::euclidean_vector{0.0, 1.0, 2.0};
auto lf = static_cast&lt;std::list&lt;double&gt;&gt;(a);
</code></pre>
    </td>
    <td>N/A</td>
  </tr>
</table>

### 4. Member Functions

<table border="1" cellspacing="0" cellpadding="5">
  <thead>
    <th>Prototype</th>
    <th>Description</th>
    <th>Usage</th>
    <th>Exception: Why thrown & what message</th>
  </thead>
  <tr>
    <td><code>double at(int) const</code></td>
    <td>Returns the value of the magnitude in the dimension given as the function parameter</td>
    <td><code>a.at(1);</code></td>
    <td>
      <b>When</b>: For Input X: when X is &lt 0 or X is &gt= number of dimensions<br />
      <b>Throw</b>: "Index X is not valid for this euclidean_vector object"
    </td>
  </tr>
  <tr>
    <td><code>double& at(int)</code></td>
    <td>Returns the reference of the magnitude in the dimension given as the function parameter</td>
    <td><code>a.at(1);</code></td>
    <td>
      <b>When</b>: For Input X: when X is &lt 0 or X is &gt= number of dimensions<br />
      <b>Throw</b>: "Index X is not valid for this euclidean_vector object"
    </td>
  </tr>
  <tr>
    <td><code>int dimensions()</code></td>
    <td>Return the number of dimensions in a particular euclidean_vector</td>
    <td><code>a.dimensions();</code></td>
    <td>N/A</td>
  </tr>
</table>

### 5. Friends

In addition to the operations indicated earlier, the following operations should be supported as
friend functions. Note that these friend operations don't modify any of the given operands.

<table border="1" cellspacing="0" cellpadding="5">
  <thead>
    <th>Name</th>
    <th>Operator</th>
    <th>Description</th>
    <th>Usage</th>
    <th>Exception: Why thrown & what message</th>
  </thead>
  <tr>
    <td>Equal</td>
    <td><code>bool operator==(euclidean_vector const&, euclidean_vector const&)</code></td>
    <td>
      True if the two vectors are equal in the number of dimensions and the magnitude in each
      dimension is equal.
    </td>
    <td><pre><code>a == b;</code></pre></td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Not Equal</td>
    <td><code>bool operator!=(euclidean_vector const&, euclidean_vector const&)</code></td>
    <td>
      True if the two vectors are not equal in the number of dimensions or the magnitude in each
      dimension is not equal.
    </td>
    <td><pre><code>a != b;</code></pre></td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>Addition</td>
    <td><code>euclidean_vector operator+(euclidean_vector const&, euclidean_vector const&)</code></td>
    <td>For adding vectors of the same dimension.</td>
    <td><pre><code>a = b + c;</code></pre></td>
    <td>
      <b>Given</b>: <code>X = b.dimensions()</code>, <code>Y = c.dimensions()</code>
      <b>When</b>: <code>X != Y</code><br />
      <b>Throw</b>: "Dimensions of LHS(X) and RHS(Y) do not match"
    </td>
  </tr>
  <tr>
    <td>Subtraction</td>
    <td><code>euclidean_vector operator-(euclidean_vector const&, euclidean_vector const&)</code></td>
    <td>For substracting vectors of the same dimension.</td>
    <td><pre><code>a = b - c;</code></pre></td>
    <td>
      <b>Given</b>: <code>X = b.dimensions()</code>, <code>Y = c.dimensions()</code>
      <b>When</b>: <code>X != Y</code><br />
      <b>Throw</b>: "Dimensions of LHS(X) and RHS(Y) do not match"
    </td>
  </tr>
  <tr>
    <td>Multiply</td>
    <td><code>euclidean_vector operator*(euclidean_vector const&, double)</code></td>
    <td>
      For scalar multiplication, e.g. <code>[1 2] * 3 = 3 * [1 2] = [3 6]</code>.
      Hint: you'll need two operators, as the scalar can be either side of the vector.
    </td>
    <td><pre><code>(1) a = b * 3;
(2) a = 3 * b;</code></pre>
    </td>
    <td>N/A</td>
    </tr>
    <tr>
      <td>Divide</td>
      <td><code>euclidean_vector operator/(euclidean_vector const&, double)</code></td>
      <td>For scalar division, e.g. [3 6] / 2 = [1.5 3]</td>
      <td>
        <pre><code>auto b = comp6771::euclidean_vector(3, 3.0);
double c;
auto a = b / c;</code></pre>
      </td>
      <td>
        <b>When</b>: <code>c == 0</code><br />
        <b>Throw</b>: "Invalid vector division by 0"
      </td>
    </tr>
    <tr>
      <td>Output Stream</td>
      <td><code>std::ostream& operator&lt;&lt;(std::ostream&, euclidean_vector const&)</code></td>
      <td>
        Prints out the magnitude in each dimension of the Euclidean vector (surrounded by
        <code>[</code> and <code>]</code>), e.g. for a 3-dimensional vector: <code>[1 2 3]</code>. Note: When printing the magnitude, simple use the double &lt;&lt; operator.
      </td>
      <td>
        <pre><code>std::cout &lt;&lt; a;
fmt::format("{}", a); // you'll need to include &lt;fmt/ostream.h&gt; for this</code></pre>
      </td>
      <td>N/A</td>
    </tr>
</table>

### 6. Utility functions

The following are functions that operate on a Euclidean vector, but shouldn't be a part of its
interface. They _may_ be friends, if you need access to the implementation, but you should avoid
friendship if you can.

<table>
  <thead>
    <th>Name</th>
    <th>Operator</th>
    <th>Description</th>
    <th>Usage</th>
    <th>Exception: Why thrown & what message</th>
  </thead>
  <tr>
    <td><code>auto euclidean_norm(euclidean_vector const& v) -&gt; double;</code></td>
    <td>
      Returns the Euclidean norm of the vector as a <code>double</code>. The Euclidean norm is the
      square root of the sum of the squares of the magnitudes in each dimension. E.g, for the vector
      <code>[1 2 3]</code> the Euclidean norm is <code>sqrt(1*1 + 2*2 + 3*3) = 3.74</code>.
    </td>
    <td><pre><code>comp6771::euclidean_norm(a);</code></pre></td>
    <td>
      <b>When</b>: <code>v.dimensions() == 0</code><br />
      <b>Throw</b>: "euclidean_vector with no dimensions does not have a norm"
    </td>
  </tr>
  <tr>
    <td><code>auto unit(euclidean_vector const& v) -&gt; euclidean_vector;</code></td>
    <td>
      Returns a Euclidean vector that is the unit vector of <code>v</code>. The magnitude for each
      dimension in the unit vector is the original vector's magnitude divided by the Euclidean norm.
    </td>
    <td><pre><code>comp6771::unit(a);</code></pre></td>
    <td>
      <b>When</b>: <code>v.dimensions() == 0</code><br />
      <b>Throw</b>: "euclidean_vector with no dimensions does not have a unit vector"
      <hr />
      <b>When</b>: <code>comp6771::euclidean_norm(v) == 0</code><br />
      <b>Throw</b>: "euclidean_vector with zero euclidean normal does not have a unit vector"
    </td>
  </tr>
  <tr>
    <td>
      <code>auto dot(euclidean_vector const& x, euclidean_vector const& y) -&gt; double</code>
    </td>
    <td>
      Computes the dot product of <code>x</code> &sdot; <code>y</code>; returns a
      <code>double</code>. E.g., <code>[1 2] &sdot; [3 4] = 1 * 3 + 2 * 4 = 11</code>
    </td>
    <td><pre><code>auto c = double{comp6771::dot(a, b)};</code></pre></td>
    <td>
      <b>Given</b>: <code>X = a.dimensions()</code>, <code>Y = b.dimensions()</code>
      <b>When</b>: <code>X != Y</code><br />
      <b>Throw</b>: "Dimensions of LHS(X) and RHS(Y) do not match"
      <hr />
      <b>Note:</b> We will not be testing the case of multiplying two 0-dimension vectors together.
    </td>
  </tr>
</table>

The Euclidean norm should only be calculated when required and ideally should be cached if required
again. We may run test cases on large vectors calculating the Euclidean norm many times. Hint:
consider using a mutable data member where appropriate in conjunction with another data member to
appropriate cache the euclidean norm. This is done for performance reasons.

### 7. Compulsory Data Members

Your Euclidean vector is **required** to store the magnitudes of each dimension inside of a
`unique_ptr`. This is a `unique_ptr` to a C-style `double` array.

To create a dynamically allocated C-style double array and add it to a unique pointer, but not
require any *direct* use of the `new`/`std::malloc` call, you can use the following:

```cpp
// ass2 spec requires we use double[]
// NOLINTNEXTLINE(modernize-avoid-c-arrays)
auto magnitudes_ = std::make_unique<double[]>(8); // 8 is an example
```

Please note, the theory for <code>unique_ptr</code> will be covered in week 5. Until that point, there will be *parts* of the assignment (e.g. move constructors, copy constructors) that you may struggle to implement. However, before week 5 lectures you are able to implement many other functions. That is because the <code>unique_ptr</code> is in many ways an alias for a raw pointer - i.e. you can treat <code>magnitudes_</code> like a raw pointer

For example:

```cpp
this->magnitudes_[0] += other.magnitudes_[0]
```

### 8. Throwing Exceptions

You are required to throw exceptions in certain cases. These are specified in the tables above. We
have provided a `euclidean_vector` exception class for you to throw. You are welcome to throw other
exceptions if you feel they are more appropriate.

**Note:** while the particular exception thrown does not matter, you are required to pass the
strings specified in the tables above. In these strings, please use common sense to substitute values like X and Y for their actual numerical values

### 9. Other notes

You must:

* Include a header guard in `euclidean_vector.h`
* Use C++20 style and methods where appropriate
* Make sure that *all appropriate member functions* are `const`-qualified
* Leave a moved-from object in a state with `0` dimensions
* Implement this class within the **comp6771** namespace
* Must assume that addition, subtraction, multiplication, and division operations on two 0-dimension
  vectors are valid operations. In all cases the result should still be a 0-dimension vector.
* We're asking you to implement `operator!=` because you'll see it in a lot of production codebases,
  and it's important that you know how to write it correctly.

You must not:

* Write to any files that aren't provided in the repo (e.g. storing your vector data in an
  auxilliary file)
* Add a main function `euclidean_vector.cpp`

You:

* Should try to mark member functions that will not throw exceptions with `noexcept`
* Are not required to make any member function explicit unless directly asked to in the spec.

### 10. `const`-correctness

You must ensure that each operator (3.) and method (4.) appropriately either has:

* A const member function; or
* A non-const member function; or
* Both a const and non-const member function

Please think carefully about this. The function declarations intentionally do not specify their
constness, except for one exception, the `at()` operator. This has an explicit `const`
and non-`const` declaration to help you out.

In most cases you will only need a single function, but in a couple of cases you will need both a
`const` and non-`const` version.

## Testing

Here is a sample and example of Catch2 tests to write

```cpp
TEST_CASE("Creation of unit vectors") {
  SECTION("You have two identical vectors") {
    auto a = comp6771::euclidean_vector(2);
    a[0] = 1;
    a[1] = 2;
    auto b = comp6771::euclidean_vector(2);
    b[0] = 1;
    b[1] = 2;

    auto c = comp6771::unit(a);
    auto d = comp6771::unit(b);
    REQUIRE(c == d);
  }
}
```

## Getting Started

If you haven't done so already, clone this repository.

```sh
$ git clone gitlab@gitlab.cse.unsw.edu.au:z5555555/20T2-cs6771-ass2
```

(Note: Replace z5555555 with your zid)

Navigate inside the directory. You can then open vscode with `code .` (note the dot).

If you haven't done so already, clone the repository:</p>

### Running your tests

Similar to the first tutorial, you simply to have to run `Ctrl+Shift+P` and then type `Run Test` and
hit enter. VS Code will compile and run all of your tests and produce an output.


### Adding more tests

Part of your assignment mark will come from the quality and extensiveness of tests that you write.

You can add more test files to the `test/euclidean_vector/` directory. Simply copy `test/euclidean_vector/euclidean_vector_test1.cpp`
into another file in that directory.

Note, everytime that you add a new file to the `test/euclidean_vector/` directory you will need to add
another few lines to `test/CMakeLists.txt`. You can once again, simply copy the test reference for
`euclidean_vector_test1.cpp` and rename the appropriate parts. Every time you update `CMakeLists.txt` in
any repository, in VSCode you should codess `Ctrl+Shift+P` and run `Reload Window` for the changes to
take effect.



## Marking Criteria

This assignment will contribute 15% to your final mark.

The assessment for the assignment recognizes the difficulty of the task, the importance of style,
and the importance of appropriate use of programming methods (e.g. using while loops instead of a
dozen if statements).

<table class="table table-bordered table-striped">
  <tr>
    <td align=right>50%</td>
    <td>
      <b>Correctness</b><br />
      The correctness of your program will be determined automatically by tests that we will run against
      your program. You will not know the full sample of tests used prior to marking.
    </td>
  </tr>
  <tr>
    <td align=right>25%</td>
    <td>
      <b>Your tests</b><br />
      You are required to write your own tests to ensure your program works.
      You will write tests in the <code>test/</code> directory. At the top of each file you will also include a block comment to explain the rational and approach you took to writing tests. Please read the <a href="https://github.com/catchorg/Catch2/blob/master/docs/tutorial.md">Catch2 tutorial</a> or review lecture/tutorial content to see how to write tests. Tests will be marked on several
      factors. These include, but are not limited to:
      <ul>
        <li>Correctness — an incorrect test is worse than useless.</li>
        <li>
          Coverage - your tests might be great, but if they don't cover the part that ends up
          failing, they weren't much good to you.
        </li>
        <li>
          Brittleness — If you change your implementation, will the tests need to be changed (this
          generally means avoiding calling functions specific to your implementation where possible
          - ones that would be private if you were doing OOP).
        </li>
        <li>
          Clarity — If your test case failed, it should be immediately obvious what went wrong (this
          means splitting it up into appropriately sized sub-tests, amongst other things).
        </li>
      </ul>
    </td>
  </tr>
  <tr>
    <td align=right>20%</td>
    <td>
      <b>C++ best practices</b><br />
      Your adherence to good C++ best practice in lecture. This is <b>not</b> saying that if you
      conform to the style guide you will receive full marks for this section. This 20% is also
      based on how well you use modern C++ methodologies taught in this course as opposed to using
      backwards-compatible C methods. Examples include: Not using primitive arrays and not using
      pointers. We will also penalise you for standard poor practices in programming, such as having
      too many nested loops, poor variable naming, etc.
    </td>
  </tr>
  <tr>
    <td align=right>5%<td>
    <b>clang-format</b><br />
    In your project folder, run the following commands on all cpp/h files in the `source` and `test` directory.<br />
    <code>$ clang-format-11 -style=file -i /path/to/file.cpp</code>
    If, for each of these files, the program outputs nothing (i.e. is linted correctly), you will receive full marks for
    this section (5%).
    A video explaining how to use clang-format can be found <a href="https://unsw.zoom.us/rec/share/4_1tE-mo2UpJRo2V0U_UVp4BRoDAeaa8hicbrvVfxRs2wxZeMb3Au5H-75jpZZWj">HERE</a>.
  </tr>
</table>

The following actions will result in a 0/100 mark for this assignment, and in some cases a 0 for
COMP6771:

* Knowingly providing your work to anyone and it is subsequently submitted (by anyone).
* Submitting any other person's work. This includes joint work.

The lecturer may vary the assessment scheme after inspecting
the assignment submissions but it will remain broadly similar to the description above.


## Originality of Work

The work you submit must be your own work.  Submission of work partially or completely derived from
any other person or jointly written with any other person is not permitted.

The penalties for such an offence may include negative marks, automatic failure of the course and
possibly other academic discipline. Assignment submissions will be examined both automatically and
manually for such submissions.

Relevant scholarship authorities will be informed if students holding scholarships are involved in
an incident of plagiarism or other misconduct.

Do not provide or show your assignment work to any other person &mdash; apart from the teaching
staff of COMP6771.

If you knowingly provide or show your assignment work to another person for any reason, and work
derived from it is submitted, you may be penalized, even if the work was submitted without your
knowledge or consent.  This may apply even if your work is submitted by a third party unknown to
you.

Note you will not be penalized if your work has the potential to be taken without your consent or
knowledge.

## Submission

This assignment is due *Monday 13th of July, 19:59:59*.
Submit the assignment using the following comand while logged into the CSE machines:

```sh
6771 submit ass2
```

This will submit whatever is on the master branch of THIS repository (the one this README.md file is
contained in) at the moment of submission.

**Please ensure that you commit and push your local code TO your gitlab repository (called the origin remote) before submitting, otherwise your code will not be submitted**

Please ensure that you can build and run your tests successfully on the CSE machine.

## Late Submission Policy

If your assignment is submitted after this date, each hour it is late reduces the maximum mark it can achieve by 2%.

For example if an assignment you submitted with a raw awarded mark of 85% was submitted 5 hours late, the late submission would have no effect (as maximum mark would be 90%).

If the same assignment was submitted 20 hours late it would be awarded
60%, the maximum mark it can achieve at that time.


<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" lang xml:lang>
<head>
  <meta charset="utf-8" />
  <meta name="generator" content="mpark/wg21" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes" />
  <meta name="dcterms.date" content="2020-07-14" />
  <title>General Directed Weighted Graph</title>
  <link href="data:image/x-icon;base64,AAABAAIAEBAAAAEAIABoBAAAJgAAACAgAAABACAAqBAAAI4EAAAoAAAAEAAAACAAAAABACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA////AIJEAACCRAAAgkQAAIJEAACCRAAAgkQAVoJEAN6CRADegkQAWIJEAACCRAAAgkQAAIJEAACCRAAA////AP///wCCRAAAgkQAAIJEAACCRAAsgkQAvoJEAP+CRAD/gkQA/4JEAP+CRADAgkQALoJEAACCRAAAgkQAAP///wD///8AgkQAAIJEABSCRACSgkQA/IJEAP99PQD/dzMA/3czAP99PQD/gkQA/4JEAPyCRACUgkQAFIJEAAD///8A////AHw+AFiBQwDqgkQA/4BBAP9/PxP/uZd6/9rJtf/bybX/upd7/39AFP+AQQD/gkQA/4FDAOqAQgBc////AP///wDKklv4jlEa/3o7AP+PWC//8+3o///////////////////////z7un/kFox/35AAP+GRwD/mVYA+v///wD///8A0Zpk+NmibP+0d0T/8evj///////+/fv/1sKz/9bCs//9/fr//////+/m2/+NRwL/nloA/5xYAPj///8A////ANKaZPjRmGH/5cKh////////////k149/3UwAP91MQD/lmQ//86rhv+USg3/m1YA/5hSAP+bVgD4////AP///wDSmmT4zpJY/+/bx///////8+TV/8mLT/+TVx//gkIA/5lVAP+VTAD/x6B//7aEVv/JpH7/s39J+P///wD///8A0ppk+M6SWP/u2sf///////Pj1f/Nj1T/2KFs/8mOUv+eWhD/lEsA/8aee/+0glT/x6F7/7J8Rvj///8A////ANKaZPjRmGH/48Cf///////+/v7/2qt//82PVP/OkFX/37KJ/86siv+USg7/mVQA/5hRAP+bVgD4////AP///wDSmmT40ppk/9CVXP/69O////////7+/v/x4M//8d/P//7+/f//////9u7n/6tnJf+XUgD/nFgA+P///wD///8A0ppk+NKaZP/RmWL/1qNy//r07///////////////////////+vXw/9akdP/Wnmn/y5FY/6JfFvj///8A////ANKaZFTSmmTo0ppk/9GYYv/Ql1//5cWm//Hg0P/x4ND/5cWm/9GXYP/RmGH/0ppk/9KaZOjVnmpY////AP///wDSmmQA0ppkEtKaZI7SmmT60ppk/9CWX//OkVb/zpFW/9CWX//SmmT/0ppk/NKaZJDSmmQS0ppkAP///wD///8A0ppkANKaZADSmmQA0ppkKtKaZLrSmmT/0ppk/9KaZP/SmmT/0ppkvNKaZCrSmmQA0ppkANKaZAD///8A////ANKaZADSmmQA0ppkANKaZADSmmQA0ppkUtKaZNzSmmTc0ppkVNKaZADSmmQA0ppkANKaZADSmmQA////AP5/AAD4HwAA4AcAAMADAACAAQAAgAEAAIABAACAAQAAgAEAAIABAACAAQAAgAEAAMADAADgBwAA+B8AAP5/AAAoAAAAIAAAAEAAAAABACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA////AP///wCCRAAAgkQAAIJEAACCRAAAgkQAAIJEAACCRAAAgkQAAIJEAACCRAAAgkQAAIJEAAyCRACMgkQA6oJEAOqCRACQgkQAEIJEAACCRAAAgkQAAIJEAACCRAAAgkQAAIJEAACCRAAAgkQAAIJEAACCRAAA////AP///wD///8A////AIJEAACCRAAAgkQAAIJEAACCRAAAgkQAAIJEAACCRAAAgkQAAIJEAACCRABigkQA5oJEAP+CRAD/gkQA/4JEAP+CRADqgkQAZoJEAACCRAAAgkQAAIJEAACCRAAAgkQAAIJEAACCRAAAgkQAAIJEAAD///8A////AP///wD///8AgkQAAIJEAACCRAAAgkQAAIJEAACCRAAAgkQAAIJEAACCRAA4gkQAwoJEAP+CRAD/gkQA/4JEAP+CRAD/gkQA/4JEAP+CRAD/gkQAxIJEADyCRAAAgkQAAIJEAACCRAAAgkQAAIJEAACCRAAAgkQAAP///wD///8A////AP///wCCRAAAgkQAAIJEAACCRAAAgkQAAIJEAACCRAAWgkQAmIJEAP+CRAD/gkQA/4JEAP+CRAD/gkQA/4JEAP+CRAD/gkQA/4JEAP+CRAD/gkQA/4JEAJyCRAAYgkQAAIJEAACCRAAAgkQAAIJEAACCRAAA////AP///wD///8A////AIJEAACCRAAAgkQAAIJEAACCRAAAgkQAdIJEAPCCRAD/gkQA/4JEAP+CRAD/gkQA/4JEAP+CRAD/gkQA/4JEAP+CRAD/gkQA/4JEAP+CRAD/gkQA/4JEAPSCRAB4gkQAAIJEAACCRAAAgkQAAIJEAAD///8A////AP///wD///8AgkQAAIJEAACCRAAAgkQASoJEANKCRAD/gkQA/4JEAP+CRAD/g0YA/39AAP9zLgD/bSQA/2shAP9rIQD/bSQA/3MuAP9/PwD/g0YA/4JEAP+CRAD/gkQA/4JEAP+CRADUgkQAToJEAACCRAAAgkQAAP///wD///8A////AP///wB+PwAAgkUAIoJEAKiCRAD/gkQA/4JEAP+CRAD/hEcA/4BBAP9sIwD/dTAA/5RfKv+viF7/vp56/76ee/+wiF7/lWAr/3YxAP9sIwD/f0AA/4RHAP+CRAD/gkQA/4JEAP+CRAD/gkQArIJEACaBQwAA////AP///wD///8A////AIBCAEBzNAD6f0EA/4NFAP+CRAD/gkQA/4VIAP92MwD/bSUA/6N1Tv/ezsL/////////////////////////////////38/D/6V3Uv9uJgD/dTEA/4VJAP+CRAD/gkQA/4JEAP+BQwD/fUAA/4FDAEj///8A////AP///wD///8AzJRd5qBlKf91NgD/dDUA/4JEAP+FSQD/cy4A/3YyAP/PuKP//////////////////////////////////////////////////////9K7qP94NQD/ciwA/4VJAP+CRAD/fkEA/35BAP+LSwD/mlYA6v///wD///8A////AP///wDdpnL/4qx3/8KJUv+PUhf/cTMA/3AsAP90LgD/4dK+/////////////////////////////////////////////////////////////////+TYxf91MAD/dTIA/31CAP+GRwD/llQA/6FcAP+gWwD8////AP///wD///8A////ANGZY/LSm2X/4ap3/92mcP+wdT3/byQA/8mwj////////////////////////////////////////////////////////////////////////////+LYxv9zLgP/jUoA/59bAP+hXAD/nFgA/5xYAPL///8A////AP///wD///8A0ppk8tKaZP/RmWL/1p9q/9ubXv/XqXj////////////////////////////7+fD/vZyG/6BxS/+gcUr/vJuE//r37f//////////////////////3MOr/5dQBf+dVQD/nVkA/5xYAP+cWAD/nFgA8v///wD///8A////AP///wDSmmTy0ppk/9KaZP/SmWP/yohJ//jo2P//////////////////////4NTG/4JDFf9lGAD/bSQA/20kAP9kGAD/fz8S/+Xb0f//////5NG9/6txN/+LOgD/m1QA/51aAP+cWAD/m1cA/5xYAP+cWADy////AP///wD///8A////ANKaZPLSmmT/0ppk/8+TWf/Unmv//v37//////////////////////+TWRr/VwsA/35AAP+ERgD/g0UA/4JGAP9lHgD/kFga/8KXX/+TRwD/jT4A/49CAP+VTQD/n10A/5xYAP+OQQD/lk4A/55cAPL///8A////AP///wD///8A0ppk8tKaZP/SmmT/y4tO/92yiP//////////////////////8NnE/8eCQP+rcTT/ez0A/3IyAP98PgD/gEMA/5FSAP+USwD/jj8A/5lUAP+JNwD/yqV2/694Mf+HNQD/jkAA/82rf/+laBj/jT4A8v///wD///8A////AP///wDSmmTy0ppk/9KaZP/LiUr/4byY///////////////////////gupX/0I5P/+Wuev/Lklz/l1sj/308AP+QSwD/ol0A/59aAP+aVQD/k0oA/8yoh///////+fXv/6pwO//Lp3v///////Pr4f+oay7y////AP///wD///8A////ANKaZPLSmmT/0ppk/8uJSv/hvJj//////////////////////+G7l//Jhkb/0ppk/96nc//fqXX/x4xO/6dkFP+QSQD/llEA/5xXAP+USgD/yaOA///////38uv/qG05/8ijdv//////8efb/6ZpLPL///8A////AP///wD///8A0ppk8tKaZP/SmmT/zIxO/9yxh///////////////////////7dbA/8iEQf/Sm2X/0Zlj/9ScZv/eqHf/2KJv/7yAQf+XTgD/iToA/5lSAP+JNgD/yKFv/611LP+HNQD/jT8A/8qmeP+kZRT/jT4A8v///wD///8A////AP///wDSmmTy0ppk/9KaZP/Pk1n/1J5q//78+//////////////////+/fv/1aFv/8iEQv/Tm2b/0ppl/9GZY//Wn2z/1pZc/9eldf/Bl2b/kUcA/4w9AP+OQAD/lUwA/59eAP+cWQD/jT8A/5ZOAP+eXADy////AP///wD///8A////ANKaZPLSmmT/0ppk/9KZY//KiEn/8d/P///////////////////////47+f/05tm/8iCP//KiEj/yohJ/8eCP//RmGH//vfy///////n1sP/rXQ7/4k4AP+TTAD/nVoA/5xYAP+cVwD/nFgA/5xYAPL///8A////AP///wD///8A0ppk8tKaZP/SmmT/0ptl/8uLTf/aq37////////////////////////////+/fz/6c2y/961jv/etY7/6Myx//78+v//////////////////////3MWv/5xXD/+ORAD/mFQA/51ZAP+cWAD/nFgA8v///wD///8A////AP///wDSmmTy0ppk/9KaZP/SmmT/0ppk/8mFRP/s1b//////////////////////////////////////////////////////////////////////////////+PD/0JFU/7NzMv+WUQD/kUsA/5tXAP+dWQDy////AP///wD///8A////ANKaZP/SmmT/0ppk/9KaZP/Sm2X/z5NZ/8yMT//z5NX/////////////////////////////////////////////////////////////////9Ofa/8yNUP/UmGH/36p5/8yTWv+qaSD/kksA/5ROAPz///8A////AP///wD///8A0ppk5NKaZP/SmmT/0ppk/9KaZP/TnGf/zY9T/82OUv/t1sD//////////////////////////////////////////////////////+7Yw//OkFX/zI5R/9OcZ//SmmP/26V0/9ymdf/BhUf/ol8R6P///wD///8A////AP///wDSmmQ80ppk9tKaZP/SmmT/0ppk/9KaZP/TnGj/zpFW/8qJSv/dson/8uHS//////////////////////////////////Lj0//etIv/y4lL/86QVf/TnGj/0ppk/9KaZP/RmWP/05xn/9ymdfjUnWdC////AP///wD///8A////ANKaZADSmmQc0ppkotKaZP/SmmT/0ppk/9KaZP/Tm2b/0Zli/8qJSf/NjlH/16Z3/+G8mP/myKr/5siq/+G8mP/Xp3f/zY5S/8qISf/RmGH/05tm/9KaZP/SmmT/0ppk/9KaZP/SmmSm0pljINWdaQD///8A////AP///wD///8A0ppkANKaZADSmmQA0ppkQtKaZMrSmmT/0ppk/9KaZP/SmmT/0ptl/9GYYf/Nj1P/y4lL/8qISP/KiEj/y4lK/82PU//RmGH/0ptl/9KaZP/SmmT/0ppk/9KaZP/SmmTO0ppkRtKaZADSmmQA0ppkAP///wD///8A////AP///wDSmmQA0ppkANKaZADSmmQA0ppkANKaZGzSmmTu0ppk/9KaZP/SmmT/0ppk/9KaZP/SmmT/0ppk/9KaZP/SmmT/0ppk/9KaZP/SmmT/0ppk/9KaZP/SmmTw0ppkcNKaZADSmmQA0ppkANKaZADSmmQA////AP///wD///8A////ANKaZADSmmQA0ppkANKaZADSmmQA0ppkANKaZBLSmmSQ0ppk/9KaZP/SmmT/0ppk/9KaZP/SmmT/0ppk/9KaZP/SmmT/0ppk/9KaZP/SmmT/0ppklNKaZBTSmmQA0ppkANKaZADSmmQA0ppkANKaZAD///8A////AP///wD///8A0ppkANKaZADSmmQA0ppkANKaZADSmmQA0ppkANKaZADSmmQy0ppkutKaZP/SmmT/0ppk/9KaZP/SmmT/0ppk/9KaZP/SmmT/0ppkvtKaZDbSmmQA0ppkANKaZADSmmQA0ppkANKaZADSmmQA0ppkAP///wD///8A////AP///wDSmmQA0ppkANKaZADSmmQA0ppkANKaZADSmmQA0ppkANKaZADSmmQA0ppkXNKaZODSmmT/0ppk/9KaZP/SmmT/0ppk5NKaZGDSmmQA0ppkANKaZADSmmQA0ppkANKaZADSmmQA0ppkANKaZADSmmQA////AP///wD///8A////ANKaZADSmmQA0ppkANKaZADSmmQA0ppkANKaZADSmmQA0ppkANKaZADSmmQA0ppkBtKaZIbSmmTo0ppk6tKaZIrSmmQK0ppkANKaZADSmmQA0ppkANKaZADSmmQA0ppkANKaZADSmmQA0ppkANKaZAD///8A////AP/8P///+B///+AH//+AAf//AAD//AAAP/AAAA/gAAAHwAAAA8AAAAPAAAADwAAAA8AAAAPAAAADwAAAA8AAAAPAAAADwAAAA8AAAAPAAAADwAAAA8AAAAPAAAADwAAAA+AAAAfwAAAP/AAAP/8AAP//gAH//+AH///4H////D//" rel="icon" />
  <!--[if lt IE 9]>
    <script src="//cdnjs.cloudflare.com/ajax/libs/html5shiv/3.7.3/html5shiv-printshiv.min.js"></script>
  <![endif]-->

</head>
<body>
<div class="wrapper">
<header id="title-block-header">
<h1 class="title" style="text-align:center">General Directed Weighted Graph</h1>

<table style="border:none;float:right">
  <tr>
    <td>Date:</td>
    <td>2020-07-14</td>
  </tr>
</table>

</header>
<div style="clear:both">
<div id="TOC" role="doc-toc">
<h1 id="toctitle">Contents</h1>
<ul>
<li><a href="#change-log"><span class="toc-section-number">1</span> Change Log<span></span></a></li>
<li><a href="#the-task"><span class="toc-section-number">2</span> The Task<span></span></a>
<ul>
<li><a href="#1-change-log"><span class="toc-section-number">1</span> Change Log<span></span></a></li>
<li><a href="#2-the-task"><span class="toc-section-number">2</span> The Task<span></span></a>
<li><a href="#21-definitions-gdwgdefinitions"><span class="toc-section-number">2.1</span> Definitions [gdwg.definitions]<span></span></a></li>
<li><a href="#22-constructors-gdwgctor"><span class="toc-section-number">2.2</span> Constructors [gdwg.ctor]<span></span></a></li>
<li><a href="#23-modifiers-gdwgmodifiers"><span class="toc-section-number">2.3</span> Modifiers [gdwg.modifiers]<span></span></a></li>
<li><a href="#24-accessors-gdwgaccessors"><span class="toc-section-number">2.4</span> Accessors [gdwg.accessors]<span></span></a></li>
<li><a href="#25-range-access-gdwgrangeaccess"><span class="toc-section-number">2.5</span> Range access [gdwg.range.access]<span></span></a></li>
<li><a href="#26-comparisons-gdwgcmp"><span class="toc-section-number">2.6</span> Comparisons [gdwg.cmp]<span></span></a></li>
<li><a href="#27-extractor-gdwgio"><span class="toc-section-number">2.7</span> Extractor [gdwg.io]<span></span></a></li>
<li><a href="#28-iterator-gdwgiterator"><span class="toc-section-number">2.8</span> Iterator [gdwg.iterator]<span></span></a>
<ul>
<li><a href="#281-iterator-constructor-gdwgiteratorctor"><span class="toc-section-number">2.8.1</span> Iterator constructor [gdwg.iterator.ctor]<span></span></a></li>
<li><a href="#282-iterator-source-gdwgiteratorsource"><span class="toc-section-number">2.8.2</span> Iterator source [gdwg.iterator.source]<span></span></a></li>
<li><a href="#283-iterator-traversal-gdwgiteratortraversal"><span class="toc-section-number">2.8.3</span> Iterator traversal [gdwg.iterator.traversal]<span></span></a></li>
<li><a href="#284-iterator-comparison-gdwgiteratorcomparison"><span class="toc-section-number">2.8.4</span> Iterator comparison [gdwg.iterator.comparison]<span></span></a></li>
</ul></li>
<li><a href="#29-compulsory-internal-representation-gdwginternal"><span class="toc-section-number">2.9</span> Compulsory internal representation [gdwg.internal]<span></span></a>
<ul>
<li><a href="#291-but-why-smart-pointers-gdwginternalrationale"><span class="toc-section-number">2.9.1</span> But why smart pointers [gdwg.internal.rationale]<span></span></a></li>
</ul></li>
<li><a href="#210-other-notes-othernotes"><span class="toc-section-number">2.10</span> Other notes [other.notes]<span></span></a>
<ul>
<li><a href="#2101-const-correctness-constcorrectness"><span class="toc-section-number">2.10.1</span> <code class="sourceCode default">const</code>-correctness [const.correctness]<span></span></a></li>
</ul></li>
</ul></li>
<li><a href="#3-getting-started"><span class="toc-section-number">3</span> Getting Started<span></span></a>
<ul>
<li><a href="#31-running-your-tests"><span class="toc-section-number">3.1</span> Running your tests<span></span></a></li>
<li><a href="#32-adding-more-tests"><span class="toc-section-number">3.2</span> Adding more tests<span></span></a></li>
</ul></li>
<li><a href="#4-marking-criteria"><span class="toc-section-number">4</span> Marking Criteria<span></span></a></li>
<li><a href="#5-originality-of-work"><span class="toc-section-number">5</span> Originality of Work<span></span></a></li>
<li><a href="#6-submission"><span class="toc-section-number">6</span> Submission<span></span></a></li>
<li><a href="#7-late-submission-policy"><span class="toc-section-number">7</span> Late Submission Policy<span></span></a></li>
<li><a href="#8-cmake-tools-workaround"><span class="toc-section-number">8</span> CMake Tools workaround<span></span></a></li>
</ul>
</div>
<p><strong>Please <code class="sourceCode default">git pull</code> daily to get the latest changes.</strong></p>
<p><strong>You will be required to update the toolchain for this assignment. See Tutorial 8 for directions.</strong></p>
<h1 data-number="1" id="change-log"><span class="header-section-number">1</span> Change Log<a href="#change-log" class="self-link"></a></h1>
<ul>
<li><strong>2020-08-01</strong>:
<ul>
<li>Removes complexity requirement for <code>erase_node</code>, as we've observed great difficulty with this specific requirement. As this course isn't a course on data structures and algorithms, you don't need to place too much stock in the complexity requirements, and we won't be marking marking too harshly if you fall outside of their bounds. We instead provided these requirements as a way to guide you toward certain containers and algorithms.</li>
<li>A student identified a typo in the iterator's specification if it is defined outside the graph class definition, which we've now corrected. If you've defined your iterator class inside the graph class, <strong>you have nothing to worry about</strong>. If you defined your iterator outside of the class definition, and it already compiles on VLAB, <strong>you have nothing to worry about</strong>, although to prevent any potential issues with automarking, we recommend you double check it matches [graph.iterator]. <strong>If you forget to do this step we'll fix it for you later at no penalty</strong>.</li>
</ul>
</ul>
<ul>
<li><strong>2020-07-28</strong>:
<ul>
<li>Clarifies typos</li>
<li>Rewords [gdwg.internal] to make edge/weight issue clearer.</li>
</ul></li>
<li><strong>2020-07-25</strong>:
<ul>
<li>Changes <code class="sourceCode default">#include &quot;gdwg/graph.h&quot;</code> to <code class="sourceCode default">#include &quot;gdwg/graph.hpp&quot;</code> in <code class="sourceCode default">source/client.cpp</code>.</li>
<li>Corrects <code class="sourceCode default">iterator::operator*</code> return expression.</li>
<li>Emboldens the note advising students <strong>not</strong> to use <code class="sourceCode default">ranges::make_common_tuple</code>.</li>
<li>Adds postconditions to <code class="sourceCode default">erase_*</code> member functions (note: these postconditions were always there, but some students asked we make it more explicit. This shouldn’t change anything that you need to do).</li>
<li>Adds advisory notes to all member functions that might throw. <strong>Although we don’t expect you’ll need to change any code as a result of this message, you should double check this for yourself.</strong></li>
<li>Updates exception messages, replacing the <code class="sourceCode default">comp6771::</code> qualification typo with <code class="sourceCode default">gdwg::</code>. <strong>This is a semantic change that you will need to action.</strong></li>
</ul></li>
<li><strong>2020-07-22</strong>:
<ul>
<li>Removes <em>mandatory</em> <code class="sourceCode default">noexcept</code> specifier from <code class="sourceCode default">is_connected</code>, <code class="sourceCode default">is_node</code>, <code class="sourceCode default">nodes</code>, and <code class="sourceCode default">find</code> (you can still add them if you think they’re necessary).</li>
<li>Removes confusing “diamond suffix” from <code class="sourceCode default">operator&lt;&lt;</code></li>
<li>Clarifies complexity for range-based <code class="sourceCode default">erase_edge</code>, to make it clearer that the complexity is with respect to the range passed in (and the graph itself is just a really big constant).</li>
<li>Changes exception messages for <code class="sourceCode default">replace_node</code> and <code class="sourceCode default">connections</code> so that they match the member functions’ names.
<table>
<thead>
<th>
Member function
</th>
<th>
Before
</th>
<th>
After
</th>
</thead>
<tr>
<td>
<code>replace_node</code>
</td>
<td>
<code>“Cannot call comp6771::graph&lt;N, E&gt;::replace on a node that doesn&#39;t exist”</code>
</td>
<td>
<code>“Cannot call comp6771::graph&lt;N, E&gt;::<b>replace_node</b> on a node that doesn&#39;t exist”</code>
</td>
</tr>
<tr>
<td>
<code>connections</code>
</td>
<td>
<code>“Cannot call gdwg::graph&lt;N, E&gt;::connected if src doesn&#39;t exist in the graph”</code>
</td>
<td>
<code>“Cannot call gdwg::graph&lt;N, E&gt;::<b>connections</b> if src doesn&#39;t exist in the graph”</code>
</td>
</tr>
</table></li>
</ul></li>
<li><strong>2020-07-20</strong>: Updates complexity for <code class="sourceCode default">erase_edge</code> and updates <code class="sourceCode default">cmake-kits.json</code> to work around vcpkg bug.</li>
<li><strong>2020-07-19</strong>: Updates complexity for <code class="sourceCode default">erase_node</code> and <code class="sourceCode default">erase_edges</code></li>
<li><strong>2020-07-18</strong>: Implements workaround for CMake Tools configure bug</li>
<li>First revision</li>
</ul>
<h1 data-number="2" id="the-task"><span class="header-section-number">2</span> The Task<a href="#the-task" class="self-link"></a></h1>
<p>Write a <code class="sourceCode default">graph</code> library type in C++, in <code class="sourceCode default">include/gdwg/graph.hpp</code>.</p>
<p>In this assignment, you will write a <em>generic directed weighted graph</em> (GDWG) with value-semantics in C++. Both the data stored at a node and the weight stored at an edge will be parameterised types. The types may be different. For example, here is a graph with nodes storing <code class="sourceCode default">std::string</code> and edges weighted by <code class="sourceCode default">int</code>:</p>
<div class="sourceCode" id="cb1"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb1-1"><a href="#cb1-1" aria-hidden="true"></a><span class="kw">using</span> graph <span class="op">=</span> gdwg<span class="op">::</span>graph<span class="op">&lt;</span>std<span class="op">::</span>string, <span class="dt">int</span><span class="op">&gt;</span>;</span></code></pre></div>
<p>Formally, this directed weighted graph <span class="math inline"><em>G</em> = (<em>N</em>, <em>E</em>)</span> will consist of a set of nodes <span class="math inline"><em>N</em></span> and a set of weighted edges <span class="math inline"><em>E</em></span>.</p>
<p>All nodes are unique, that is to say, no two nodes will have the same value and shall not compare equal using <code class="sourceCode default">operator==</code>.</p>
<p>Given a node, an edge directed into it is called an <em>incoming edge</em> and an edge directed out of it is called an <em>outgoing edge</em>. The <em>in-degree</em> of a node is the number of its incoming edges. Similarly, the <em>out-degree</em> of a node is the number of its outgoing edges. Given a directed edge from <code class="sourceCode default">src</code> to <code class="sourceCode default">dst</code>, <code class="sourceCode default">src</code> is the <em>source node</em> and <code class="sourceCode default">dst</code> is known as the <em>destination node</em>.</p>
<p>Edges can be reflexive, that is to say, the source and destination nodes of an edge could be the same.</p>
<p><span class="math inline"><em>G</em></span> is a multi-edged graph, as there may be two edges from the same source node to the same destination node with two different weights. Two edges from the same source node to the same destination node cannot have the same weight.</p>
<h2 data-number="2.1" id="definitions-gdwg.definitions"><span class="header-section-number">2.1</span> Definitions [gdwg.definitions]<a href="#definitions-gdwg.definitions" class="self-link"></a></h2>
<ol type="1">
<li><p>Some words have special meaning in this document. This section precisicely defines those words.</p>
<ul>
<li><em>Preconditions</em>: the conditions that the function assumes to hold whenever it is called; violation of any preconditions results in undefined</li>
<li><em>Effects</em>: the actions performed by the function.</li>
<li><em>Postconditions</em>: the conditions (sometimes termed observable results) established by the function.</li>
<li><em>Returns</em>: a description of the value(s) returned by the function.</li>
<li><em>Throws</em>: any exceptions thrown by the function, and the conditions that would cause the exception.</li>
<li><em>Complexity</em>: the time and/or space complexity of the function.</li>
<li><em>Remarks</em>: additional semantic constraints on the function.</li>
<li><em>Unspecified</em>: the implementation is allowed to make its own decisions regarding what is unspecified, provided that it still follows the explicitly specified wording.</li>
</ul></li>
<li><p>An <em>Effects</em> element may specify semantics for a function <code class="sourceCode default">F</code> in code using the term <em>Equivalent to</em>. The semantics for <code class="sourceCode default">F</code> are interpreted as follows:</p>
<ul>
<li>All of the above terminology applies to the provided code, whether or not it is explicitly specified. [<em>Example</em>: If <code class="sourceCode default">F</code> has a <em>Preconditions</em> element, but the code block doesn’t explicitly check them, then it is implied that the preconditions have been checked. —<em>end example</em>]</li>
<li>If there is not a <em>Returns</em> element, and <code class="sourceCode default">F</code> has a non-<code class="sourceCode default">void</code> return type, all the return statements are in the code block.</li>
<li><em>Throws</em>, <em>Postconditions</em>, and <em>Complexity</em> elements always have priority over the code block.</li>
</ul></li>
<li><p>Specified complexity requirements are upper bounds, and implementations that provide better complexity guarantees meet the requirements.</p></li>
<li><p>The class synopsis is the minimum text your header requires to compile most tests (this doesn’t mean that it will necessarily link or run as expected).</p></li>
<li><p>Blue text in code will link to C++ Reference or to another part of this document.</p></li>
<li><p>This section makes use of [stable.names]. A stable name is a short name for a (sub)section, and isn’t supposed to change. We will use these to reference specific sections of the document. [<em>Example</em>:</p></li>
</ol>
<blockquote>
<p>Student: Do we need to define <code class="sourceCode default">gdwg::graph&lt;N, E&gt;::operator!=</code>?</p>
<p>Tutor: [other.notes] mentions that you don’t need to so you can get used to C++20’s generated operators.</p>
</blockquote>
<p>—<em>end example</em>]</p>
<h2 data-number="2.2" id="constructors-gdwg.ctor"><span class="header-section-number">2.2</span> Constructors [gdwg.ctor]<a href="#constructors-gdwg.ctor" class="self-link"></a></h2>
<p><strong>It’s very important your constructors work. If we can’t validly construct your objects, we can’t test any of your other functions.</strong></p>
<div class="sourceCode" id="cb2"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb2-1"><a href="#cb2-1" aria-hidden="true"></a>graph<span class="op">()</span>;</span></code></pre></div>
<ol type="1">
<li><p><em>Effects</em>: <a href="https://en.cppreference.com/w/cpp/language/value_initialization">Value initialises</a> all members.</p></li>
<li><p><em>Throws</em>: Nothing.</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb3"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb3-1"><a href="#cb3-1" aria-hidden="true"></a>graph<span class="op">(</span>std<span class="op">::</span>initializer_list<span class="op">&lt;</span>N<span class="op">&gt;</span> il<span class="op">)</span>;</span></code></pre></div>
<ol start="3" type="1">
<li><em>Effects</em>: Equivalent to: <code class="sourceCode default">graph(il.begin(), il.end());</code></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb4"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb4-1"><a href="#cb4-1" aria-hidden="true"></a><span class="kw">template</span><span class="op">&lt;</span>ranges<span class="op">::</span>forward_iterator I, ranges<span class="op">::</span>sentinel_for<span class="op">&lt;</span>I<span class="op">&gt;</span> S<span class="op">&gt;</span></span>
<span id="cb4-2"><a href="#cb4-2" aria-hidden="true"></a><span class="kw">requires</span> ranges<span class="op">::</span>indirectly_copyable<span class="op">&lt;</span>I, N<span class="op">*&gt;</span></span>
<span id="cb4-3"><a href="#cb4-3" aria-hidden="true"></a>graph<span class="op">(</span>I first, S last<span class="op">)</span>;</span></code></pre></div>
<ol start="4" type="1">
<li><em>Effects</em>: Initialises the graph’s node collection with the range <code class="sourceCode default">[first, last)</code>.</li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb5"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb5-1"><a href="#cb5-1" aria-hidden="true"></a><span class="kw">template</span><span class="op">&lt;</span>ranges<span class="op">::</span>forward_iterator I, ranges<span class="op">::</span>sentinel_for<span class="op">&lt;</span>I<span class="op">&gt;</span> S<span class="op">&gt;</span></span>
<span id="cb5-2"><a href="#cb5-2" aria-hidden="true"></a><span class="kw">requires</span> ranges<span class="op">::</span>indirectly_copyable<span class="op">&lt;</span>I, value_type<span class="op">*&gt;</span></span>
<span id="cb5-3"><a href="#cb5-3" aria-hidden="true"></a>graph<span class="op">(</span>I first, S last<span class="op">)</span>;</span></code></pre></div>
<ol start="5" type="1">
<li><em>Effects</em>: Initialises the graph’s node and edge collections with the range <code class="sourceCode default">[first, last)</code>.</li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb6"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb6-1"><a href="#cb6-1" aria-hidden="true"></a>graph<span class="op">(</span>graph<span class="op">&amp;&amp;</span> other<span class="op">)</span> <span class="kw">noexcept</span>;</span></code></pre></div>
<ol start="6" type="1">
<li><em>Postconditions</em>:</li>
</ol>
<ul>
<li><code class="sourceCode default">*this</code> is equal to the value <code class="sourceCode default">other</code> had before this constructor’s invocation.</li>
<li><code class="sourceCode default">other.empty()</code> is <code class="sourceCode default">true</code>.</li>
<li>All iterators pointing to elements owned by <code class="sourceCode default">*this</code> prior to this constructor’s invocation are invalidated.</li>
<li>All iterators pointing to elements owned by <code class="sourceCode default">other</code> prior to this constructor’s invocation remain valid, but now point to the elements owned by <code class="sourceCode default">*this</code>.</li>
</ul>
<p><br /></p>
<div class="sourceCode" id="cb7"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb7-1"><a href="#cb7-1" aria-hidden="true"></a><span class="kw">auto</span> <span class="kw">operator</span><span class="op">=(</span>graph<span class="op">&amp;&amp;</span> other<span class="op">)</span> <span class="kw">noexcept</span> <span class="op">-&gt;</span> graph<span class="op">&amp;</span>;</span></code></pre></div>
<ol start="7" type="1">
<li><em>Effects</em>: All existing nodes and edges are either move-assigned to, or are destroyed.</li>
<li><em>Postconditions</em>:</li>
</ol>
<ul>
<li><code class="sourceCode default">*this</code> is equal to the value <code class="sourceCode default">other</code> had before this operator’s invocation.</li>
<li><code class="sourceCode default">other.empty()</code> is <code class="sourceCode default">true</code>.</li>
<li>All iterators pointing to elements owned by <code class="sourceCode default">*this</code> prior to this operator’s invocation are invalidated.</li>
<li>All iterators pointing to elements owned by <code class="sourceCode default">other</code> prior to this operator’s invocation remain valid, but now point to the elements owned by <code class="sourceCode default">*this</code>.</li>
</ul>
<ol start="9" type="1">
<li><em>Returns</em>: <code class="sourceCode default">*this</code>.</li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb8"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb8-1"><a href="#cb8-1" aria-hidden="true"></a>graph<span class="op">(</span>graph <span class="kw">const</span><span class="op">&amp;</span> other<span class="op">)</span>;</span></code></pre></div>
<ol start="10" type="1">
<li><em>Postconditions</em>: <code class="sourceCode default">*this == other</code> is <code class="sourceCode default">true</code>.</li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb9"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb9-1"><a href="#cb9-1" aria-hidden="true"></a><span class="kw">auto</span> <span class="kw">operator</span><span class="op">=(</span>graph <span class="kw">const</span><span class="op">&amp;</span> other<span class="op">)</span> <span class="op">-&gt;</span> graph<span class="op">&amp;</span>;</span></code></pre></div>
<ol start="11" type="1">
<li><p><em>Postconditions</em>:</p>
<ul>
<li><code class="sourceCode default">*this == other</code> is <code class="sourceCode default">true</code>.</li>
<li>All iterators pointing to elements owned by <code class="sourceCode default">*this</code> prior to this operator’s invocation are invalidated.</li>
</ul></li>
<li><p><em>Returns</em>: <code class="sourceCode default">*this</code>.</p></li>
</ol>
<h2 data-number="2.3" id="modifiers-gdwg.modifiers"><span class="header-section-number">2.3</span> Modifiers [gdwg.modifiers]<a href="#modifiers-gdwg.modifiers" class="self-link"></a></h2>
<div class="sourceCode" id="cb10"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb10-1"><a href="#cb10-1" aria-hidden="true"></a><span class="kw">auto</span> insert_node<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> value<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span></code></pre></div>
<ol type="1">
<li><p><em>Effects</em>: Adds a new node with value <code class="sourceCode default">value</code> to the graph if, and only if, there is no node equivalent to <code class="sourceCode default">value</code> already stored.</p></li>
<li><p><em>Postconditions</em>: All iterators are invalidated.</p></li>
<li><p><em>Returns</em>: <code class="sourceCode default">true</code> if the node is added to the graph and <code class="sourceCode default">false</code> otherwise.</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb11"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb11-1"><a href="#cb11-1" aria-hidden="true"></a><span class="kw">auto</span> insert_edge<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> src, N <span class="kw">const</span><span class="op">&amp;</span> dst, E <span class="kw">const</span><span class="op">&amp;</span> weight<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span></code></pre></div>
<ol start="4" type="1">
<li><p><em>Effects</em>: Adds a new edge representing <code class="sourceCode default">src</code> → <code class="sourceCode default">dst</code> with weight <code class="sourceCode default">weight</code>, if, and only if, there is no edge equivalent to <code class="sourceCode default">value_type{src, dst, weight}</code> already stored. [<em>Note</em>:⁠ Nodes are allowed to be connected to themselves. —<em>end note</em>]</p></li>
<li><p><em>Postconditions</em>: All iterators are invalidated.</p></li>
<li><p><em>Returns</em>: <code class="sourceCode default">true</code> if the node is added to the graph and <code class="sourceCode default">false</code> otherwise.</p></li>
<li><p><em>Throws</em>: <code class="sourceCode default">std::runtime_error(&quot;Cannot call gdwg::graph&lt;N, E&gt;::insert_edge when either src or dst node does not exist&quot;)</code> if either of <code class="sourceCode default">is_node(src)</code> or <code class="sourceCode default">is_node(dst)</code> are <code class="sourceCode default">false</code>. [<em>Note</em>: Unlike Assignment 2, the exception message must be used verbatim. —<em>end note</em>]</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb12"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb12-1"><a href="#cb12-1" aria-hidden="true"></a><span class="kw">auto</span> replace_node<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> old_data, N <span class="kw">const</span><span class="op">&amp;</span> new_data<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span></code></pre></div>
<ol start="8" type="1">
<li><p><em>Effects</em>: Replaces the original data, <code class="sourceCode default">old_data</code>, stored at this particular node by the replacement data, <code class="sourceCode default">new_data</code>. Does nothing if <code class="sourceCode default">new_data</code> already exists as a node.</p></li>
<li><p><em>Postconditions</em>: All iterators are invalidated.</p></li>
<li><p><em>Returns</em>: <code class="sourceCode default">false</code> if a node that contains value <code class="sourceCode default">new_data</code> already exists and <code class="sourceCode default">true</code> otherwise.</p></li>
<li><p><em>Throws</em>: <code class="sourceCode default">std::runtime_error(&quot;Cannot call gdwg::graph&lt;N, E&gt;::replace_node on a node that doesn&#39;t exist&quot;)</code> if <code class="sourceCode default">is_node(old_data)</code> is <code class="sourceCode default">false</code>. [<em>Note</em>: Unlike Assignment 2, the exception message must be used verbatim. —<em>end note</em>]</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb13"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb13-1"><a href="#cb13-1" aria-hidden="true"></a><span class="kw">auto</span> merge_replace_node<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> old_data, N <span class="kw">const</span><span class="op">&amp;</span> new_data<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">void</span>;</span></code></pre></div>
<ol start="12" type="1">
<li><p><em>Effects</em>: The node equivalent to <code class="sourceCode default">old_data</code> in the graph are replaced with instances of <code class="sourceCode default">new_data</code>. After completing, every incoming and outgoing edge of <code class="sourceCode default">old_data</code> becomes an incoming/ougoing edge of <code class="sourceCode default">new_data</code>, except that duplicate edges shall be removed.</p></li>
<li><p><em>Postconditions</em>: All iterators are invalidated.</p></li>
<li><p><em>Throws</em>: <code class="sourceCode default">std::runtime_error(&quot;Cannot call gdwg::graph&lt;N, E&gt;::merge_replace_node on old or new data if they don&#39;t exist in the graph&quot;)</code> if either of <code class="sourceCode default">is_node(old_data)</code> or <code class="sourceCode default">is_node(new_data)</code> are <code class="sourceCode default">false</code>. [<em>Note</em>: Unlike Assignment 2, the exception message must be used verbatim. —<em>end note</em>]</p></li>
<li><p>[<em>Note</em>: The following examples use the format <span class="math inline">(<em>N</em><sub><em>s</em><em>r</em><em>c</em></sub>, <em>N</em><sub><em>d</em><em>s</em><em>t</em></sub>, <em>E</em>)</span>. [<em>Example</em>: Basic example.</p></li>
</ol>
<ul>
<li>Operation: <code class="sourceCode default">merge_replace_node(A, B)</code></li>
<li>Graph before: <span class="math inline">(<em>A</em>, <em>B</em>, 1), (<em>A</em>, <em>C</em>, 2), (<em>A</em>, <em>D</em>, 3)</span></li>
<li>Graph after : <span class="math inline">(<em>B</em>, <em>B</em>, 1), (<em>B</em>, <em>C</em>, 2), (<em>B</em>, <em>D</em>, 3)</span></li>
</ul>
<p>—<em>end example</em>][<em>Example</em>: Duplicate edge removed example.</p>
<ul>
<li>Operation: <code class="sourceCode default">merge_replace_node(A, B)</code></li>
<li>Graph before: <span class="math inline">(<em>A</em>, <em>B</em>, 1), (<em>A</em>, <em>C</em>, 2), (<em>A</em>, <em>D</em>, 3), (<em>B</em>, <em>B</em>, 1)</span></li>
<li>Graph after : <span class="math inline">(<em>B</em>, <em>B</em>, 1), (<em>B</em>, <em>C</em>, 2), (<em>B</em>, <em>D</em>, 3)</span></li>
</ul>
<p>—<em>end example</em>][<em>Example</em>: Diagrammatic example.</p>
<p><img src="https://i.imgur.com/gCDHqrD.png" /></p>
<p>—<em>end example</em>] —<em>end note</em>]</p>
<p><br /></p>
<div class="sourceCode" id="cb14"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb14-1"><a href="#cb14-1" aria-hidden="true"></a><span class="kw">auto</span> erase_node<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> value<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span></code></pre></div>
<ol start="16" type="1">
<li><p><em>Effects</em>: Erases all nodes equivalent to <code class="sourceCode default">value</code>, including all incoming and outgoing edges.</p></li>
<li><p><em>Returns</em>: <code class="sourceCode default">true</code> if <code class="sourceCode default">value</code> was removed; <code class="sourceCode default">false</code> otherwise.</p></li>
<li><p><em>Postconditions</em>: All iterators are invalidated.</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb15"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb15-1"><a href="#cb15-1" aria-hidden="true"></a><span class="kw">auto</span> erase_edge<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> src, N <span class="kw">const</span><span class="op">&amp;</span> dst, E <span class="kw">const</span><span class="op">&amp;</span> weight<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span></code></pre></div>
<ol start="20" type="1">
<li><p><em>Effects</em>: Erases an edge representing <code class="sourceCode default">src</code> → <code class="sourceCode default">dst</code> with weight <code class="sourceCode default">weight</code>.</p></li>
<li><p><em>Returns</em>: <code class="sourceCode default">true</code> if an edge was removed; <code class="sourceCode default">false</code> otherwise.</p></li>
<li><p><em>Postconditions</em>: All iterators are invalidated.</p></li>
<li><p><em>Throws</em>: <code class="sourceCode default">std::runtime_error(&quot;Cannot call gdwg::graph&lt;N, E&gt;::erase_edge on src or dst if they don&#39;t exist in the graph&quot;)</code> if either <code class="sourceCode default">is_node(src)</code> or <code class="sourceCode default">is_node(dst)</code> is <code class="sourceCode default">false</code>. [<em>Note</em>: Unlike Assignment 2, the exception message must be used verbatim. —<em>end note</em>]</p></li>
<li><p><em>Complexity</em>: <span class="math inline"><em>O</em>(log (<em>n</em>) + <em>e</em>)</span>, where <span class="math inline"><em>n</em></span> is the total number of stored nodes and <span class="math inline"><em>e</em></span> is the total number of stored edges.</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb16"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb16-1"><a href="#cb16-1" aria-hidden="true"></a><span class="kw">auto</span> erase_edge<span class="op">(</span>iterator i<span class="op">)</span> <span class="op">-&gt;</span> iterator;</span></code></pre></div>
<ol start="25" type="1">
<li><p><em>Effects</em>: Erases the edge pointed to by <code class="sourceCode default">i</code>.</p></li>
<li><p><em>Complexity</em>: Amortised constant time.</p></li>
<li><p><em>Returns</em>: An iterator pointing to the element immediately after <code class="sourceCode default">i</code> prior to the element being erased. If no such element exists, returns <code class="sourceCode default">end()</code>.</p></li>
<li><p><em>Postconditions</em>: All iterators are invalidated. [<em>Note</em>: The postcondition is slightly stricter than a real-world container to help make the assingment easier (i.e. we won’t be testing any iterators post-erasure). —<em>end note</em>]</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb17"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb17-1"><a href="#cb17-1" aria-hidden="true"></a><span class="kw">auto</span> erase_edge<span class="op">(</span>iterator i, iterator s<span class="op">)</span> <span class="op">-&gt;</span> iterator;</span></code></pre></div>
<ol start="29" type="1">
<li><p><em>Effects</em>: Erases all edges in the range <code class="sourceCode default">[i, s)</code>.</p></li>
<li><p><em>Complexity</em> <span class="math inline"><em>O</em>(<em>d</em>)</span>, where <span class="math inline"><em>d</em>=</span><code class="sourceCode default">ranges::distance(i, s)</code>.</p></li>
<li><p><em>Returns</em>: An iterator equivalent to <code class="sourceCode default">s</code> prior to the range being erased. If no such element exists, returns <code class="sourceCode default">end()</code>.</p></li>
<li><p><em>Postconditions</em>: All iterators are invalidated. [<em>Note</em>: The postcondition is slightly stricter than a real-world container to help make the assingment easier (i.e. we won’t be testing any iterators post-erasure). —<em>end note</em>]</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb18"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb18-1"><a href="#cb18-1" aria-hidden="true"></a><span class="kw">auto</span> clear<span class="op">()</span> <span class="kw">noexcept</span> <span class="op">-&gt;</span> <span class="dt">void</span>;</span></code></pre></div>
<ol start="33" type="1">
<li><p><em>Effects</em>: Erases all nodes from the graph.</p></li>
<li><p><em>Postconditions</em>: <code class="sourceCode default">empty()</code> is <code class="sourceCode default">true</code>.</p></li>
</ol>
<h2 data-number="2.4" id="accessors-gdwg.accessors"><span class="header-section-number">2.4</span> Accessors [gdwg.accessors]<a href="#accessors-gdwg.accessors" class="self-link"></a></h2>
<div class="sourceCode" id="cb19"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb19-1"><a href="#cb19-1" aria-hidden="true"></a><span class="op">[[</span><span class="at">nodiscard</span><span class="op">]]</span> <span class="kw">auto</span> is_node<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> value<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span></code></pre></div>
<ol type="1">
<li><p><em>Returns</em>: <code class="sourceCode default">true</code> if a node equivalent to <code class="sourceCode default">value</code> exists in the graph, and <code class="sourceCode default">false</code> otherwise.</p></li>
<li><p><em>Complexity</em>: <span class="math inline"><em>O</em>(log (<em>n</em>))</span> time.</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb20"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb20-1"><a href="#cb20-1" aria-hidden="true"></a><span class="op">[[</span><span class="at">nodiscard</span><span class="op">]]</span> <span class="kw">auto</span> empty<span class="op">()</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span></code></pre></div>
<ol start="3" type="1">
<li><em>Returns</em>: <code class="sourceCode default">true</code> if there are no nodes in the graph, and <code class="sourceCode default">false</code> otherwise.</li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb21"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb21-1"><a href="#cb21-1" aria-hidden="true"></a><span class="op">[[</span><span class="at">nodiscard</span><span class="op">]]</span> <span class="kw">auto</span> is_connected<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> src, N <span class="kw">const</span><span class="op">&amp;</span> dst<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span></code></pre></div>
<ol start="4" type="1">
<li><p><em>Returns</em>: <code class="sourceCode default">true</code> if an edge <code class="sourceCode default">src</code> → <code class="sourceCode default">dst</code> exists in the graph, and <code class="sourceCode default">false</code> otherwise.</p></li>
<li><p><em>Throws</em>: <code class="sourceCode default">std::runtime_error(&quot;Cannot call gdwg::graph&lt;N, E&gt;::is_connected if src or dst node don&#39;t exist in the graph&quot;)</code> if either of <code class="sourceCode default">is_node(src)</code> or <code class="sourceCode default">is_node(dst)</code> are <code class="sourceCode default">false</code>. [<em>Note</em>: Unlike Assignment 2, the exception message must be used verbatim. —<em>end note</em>]</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb22"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb22-1"><a href="#cb22-1" aria-hidden="true"></a><span class="op">[[</span><span class="at">nodiscard</span><span class="op">]]</span> <span class="kw">auto</span> nodes<span class="op">()</span> <span class="op">-&gt;</span> std<span class="op">::</span>vector<span class="op">&lt;</span>N<span class="op">&gt;</span>;</span></code></pre></div>
<ol start="6" type="1">
<li><p><em>Returns</em>: A sequence of all stored nodes, sorted in ascending order.</p></li>
<li><p><em>Complexity</em>: <span class="math inline"><em>O</em>(<em>n</em>)</span>, where <span class="math inline"><em>n</em></span> is the number of stored nodes.</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb23"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb23-1"><a href="#cb23-1" aria-hidden="true"></a><span class="op">[[</span><span class="at">nodiscard</span><span class="op">]]</span> <span class="kw">auto</span> weights<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> src, N <span class="kw">const</span><span class="op">&amp;</span> dst<span class="op">)</span> <span class="op">-&gt;</span> std<span class="op">::</span>vector<span class="op">&lt;</span>E<span class="op">&gt;</span>;</span></code></pre></div>
<ol start="8" type="1">
<li><p><em>Returns</em>: A sequence of weights from <code class="sourceCode default">src</code> to <code class="sourceCode default">dst</code>, sorted in ascending order.</p></li>
<li><p><em>Complexity</em>: <span class="math inline"><em>O</em>(log (<em>n</em>) + <em>e</em>)</span>, where <span class="math inline"><em>n</em></span> is the number of stored nodes and <span class="math inline"><em>e</em></span> is the number of stored edges.</p></li>
<li><p><em>Throws</em>: <code class="sourceCode default">std::runtime_error(&quot;Cannot call gdwg::graph&lt;N, E&gt;::weights if src or dst node don&#39;t exist in the graph&quot;)</code> if either of <code class="sourceCode default">is_node(src)</code> or <code class="sourceCode default">is_node(dst)</code> are <code class="sourceCode default">false</code>. [<em>Note</em>: Unlike Assignment 2, the exception message must be used verbatim. —<em>end note</em>]</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb24"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb24-1"><a href="#cb24-1" aria-hidden="true"></a><span class="op">[[</span><span class="at">nodiscard</span><span class="op">]]</span> <span class="kw">auto</span> find<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> src, N <span class="kw">const</span><span class="op">&amp;</span> dst, E <span class="kw">const</span><span class="op">&amp;</span> weight<span class="op">)</span> <span class="op">-&gt;</span> iterator;</span></code></pre></div>
<ol start="11" type="1">
<li><p><em>Returns</em>: An iterator pointing to an edge equivalent to <code class="sourceCode default">value_type{src, dst, weight}</code>, or <code class="sourceCode default">end()</code> if no such edge exists.</p></li>
<li><p><em>Complexity</em>: <span class="math inline"><em>O</em>(log (<em>n</em>) + log (<em>e</em>))</span>, where <span class="math inline"><em>n</em></span> is the number of stored nodes and <span class="math inline"><em>e</em></span> is the number of stored edges.</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb25"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb25-1"><a href="#cb25-1" aria-hidden="true"></a><span class="op">[[</span><span class="at">nodiscard</span><span class="op">]]</span> <span class="kw">auto</span> connections<span class="op">(</span>N <span class="kw">const</span><span class="op">&amp;</span> src<span class="op">)</span> <span class="op">-&gt;</span> std<span class="op">::</span>vector<span class="op">&lt;</span>N<span class="op">&gt;</span>;</span></code></pre></div>
<ol start="13" type="1">
<li><p><em>Returns</em>: A sequence of nodes (found from any immediate outgoing edge) connected to <code class="sourceCode default">src</code>, sorted in ascending order, with respect to the connected nodes.</p></li>
<li><p><em>Complexity</em>: <span class="math inline"><em>O</em>(log (<em>n</em>) + <em>e</em>)</span>, where <span class="math inline"><em>e</em></span> is the number of outgoing edges associated with <code class="sourceCode default">src</code>.</p></li>
<li><p><em>Throws</em>: <code class="sourceCode default">std::runtime_error(&quot;Cannot call gdwg::graph&lt;N, E&gt;::connections if src doesn&#39;t exist in the graph&quot;)</code> if <code class="sourceCode default">is_node(src)</code> is <code class="sourceCode default">false</code>. [<em>Note</em>: Unlike Assignment 2, the exception message must be used verbatim. —<em>end note</em>]</p></li>
</ol>
<h2 data-number="2.5" id="range-access-gdwg.range.access"><span class="header-section-number">2.5</span> Range access [gdwg.range.access]<a href="#range-access-gdwg.range.access" class="self-link"></a></h2>
<div class="sourceCode" id="cb26"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb26-1"><a href="#cb26-1" aria-hidden="true"></a><span class="op">[[</span><span class="at">nodiscard</span><span class="op">]]</span> <span class="kw">auto</span> begin<span class="op">()</span> <span class="kw">const</span> <span class="op">-&gt;</span> iterator;</span></code></pre></div>
<ol type="1">
<li><em>Returns</em>: An iterator pointing to the first element in the container.</li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb27"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb27-1"><a href="#cb27-1" aria-hidden="true"></a><span class="op">[[</span><span class="at">nodiscard</span><span class="op">]]</span> <span class="kw">auto</span> end<span class="op">()</span> <span class="kw">const</span> <span class="op">-&gt;</span> iterator;</span></code></pre></div>
<ol start="2" type="1">
<li><p><em>Returns</em>: An iterator denoting the end of the range that <code class="sourceCode default">begin()</code> points to.</p></li>
<li><p><em>Remarks</em>: <code class="sourceCode default">[begin(), end())</code> shall denote a valid range.</p></li>
</ol>
<h2 data-number="2.6" id="comparisons-gdwg.cmp"><span class="header-section-number">2.6</span> Comparisons [gdwg.cmp]<a href="#comparisons-gdwg.cmp" class="self-link"></a></h2>
<div class="sourceCode" id="cb28"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb28-1"><a href="#cb28-1" aria-hidden="true"></a><span class="op">[[</span><span class="at">nodiscard</span><span class="op">]]</span> <span class="kw">auto</span> <span class="kw">operator</span><span class="op">==(</span>graph <span class="kw">const</span><span class="op">&amp;</span> other<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span></code></pre></div>
<ol type="1">
<li><p><em>Returns</em>: <code class="sourceCode default">true</code> if <code class="sourceCode default">*this</code> and <code class="sourceCode default">other</code> contain exactly the same nodes and edges, and <code class="sourceCode default">false</code> otherwise.</p></li>
<li><p><em>Complexity</em>: <span class="math inline"><em>O</em>(<em>n</em> + <em>e</em>)</span> where <span class="math inline"><em>n</em></span> is the sum of stored nodes in <code class="sourceCode default">*this</code> and <code class="sourceCode default">other</code>, and <span class="math inline"><em>e</em></span> is the sum of stored edges in <code class="sourceCode default">*this</code> and <code class="sourceCode default">other</code>.</p></li>
</ol>
<h2 data-number="2.7" id="extractor-gdwg.io"><span class="header-section-number">2.7</span> Extractor [gdwg.io]<a href="#extractor-gdwg.io" class="self-link"></a></h2>
<div class="sourceCode" id="cb29"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb29-1"><a href="#cb29-1" aria-hidden="true"></a><span class="kw">friend</span> <span class="kw">auto</span> <span class="kw">operator</span><span class="op">&lt;&lt;(</span>std<span class="op">::</span>ostream<span class="op">&amp;</span> os, graph <span class="kw">const</span><span class="op">&amp;</span> g<span class="op">)</span> <span class="op">-&gt;</span> std<span class="op">::</span>ostream<span class="op">&amp;</span>;</span></code></pre></div>
<ol type="1">
<li><p><em>Effects</em>: Behaves as a <a href="https://en.cppreference.com/w/cpp/named_req/FormattedOutputFunction">formatted output function</a> of <code class="sourceCode default">os</code>.</p></li>
<li><p><em>Returns</em>: <code class="sourceCode default">os</code>.</p></li>
<li><p><em>Remarks</em>: The format is specified thusly:</p></li>
</ol>
<div class="sourceCode" id="cb30"><pre class="sourceCode default"><code class="sourceCode default"><span id="cb30-1"><a href="#cb30-1" aria-hidden="true"></a>[source_node<span class="math inline"><sub>1</sub></span>] [edges<span class="math inline"><sub>1</sub></span>]</span>
<span id="cb30-2"><a href="#cb30-2" aria-hidden="true"></a>[source_node<span class="math inline"><sub>2</sub></span>] [edges<span class="math inline"><sub>2</sub></span>]</span>
<span id="cb30-3"><a href="#cb30-3" aria-hidden="true"></a>...</span>
<span id="cb30-4"><a href="#cb30-4" aria-hidden="true"></a>[source_node<span class="math inline"><sub><em>n</em></sub></span>] [edges<span class="math inline"><sub><em>n</em></sub></span>]</span></code></pre></div>
<p><code class="sourceCode default">[source_node<span class="math inline"><sub>1</sub></span>]</code>, …, <code class="sourceCode default">[source_node<span class="math inline"><sub><em>n</em></sub></span>]</code> are placeholders for all nodes that the graph stores, sorted in ascending order. <code class="sourceCode default">[edges<span class="math inline"><sub>1</sub></span>]</code>, …, <code class="sourceCode default">[edges<span class="math inline"><sub><em>n</em></sub></span>]</code> are placeholders for</p>
<div class="sourceCode" id="cb31"><pre class="sourceCode default"><code class="sourceCode default"><span id="cb31-1"><a href="#cb31-1" aria-hidden="true"></a>(</span>
<span id="cb31-2"><a href="#cb31-2" aria-hidden="true"></a>  [node<span class="math inline"><sub><em>n</em></sub></span>_connected_node<span class="math inline"><sub>1</sub></span>] | [weight]</span>
<span id="cb31-3"><a href="#cb31-3" aria-hidden="true"></a>  [node<span class="math inline"><sub><em>n</em></sub></span>_connected_node<span class="math inline"><sub>2</sub></span>] | [weight]</span>
<span id="cb31-4"><a href="#cb31-4" aria-hidden="true"></a>  ...</span>
<span id="cb31-5"><a href="#cb31-5" aria-hidden="true"></a>  [node<span class="math inline"><sub><em>n</em></sub></span>_connected_node<span class="math inline"><sub><em>n</em></sub></span>] | [weight]</span>
<span id="cb31-6"><a href="#cb31-6" aria-hidden="true"></a>)</span></code></pre></div>
<p>where <code class="sourceCode default">[node<span class="math inline"><sub><em>n</em></sub></span>_conencted_node<span class="math inline"><sub>1</sub></span>] | [weight]</code>, …, <code class="sourceCode default">[node<span class="math inline"><sub><em>n</em></sub></span>_connected_node<span class="math inline"><sub><em>n</em></sub></span>] | [weight]</code> are placeholders for each node’s connections and corresponding weight, also sorted in ascending order. [<em>Note</em>: If a node doesn’t have any connections, then its corresponding <code class="sourceCode default">[edges<span class="math inline"><sub><em>n</em></sub></span>]</code> should be a line-separated pair of parentheses —<em>end note</em>]</p>
<p>[<em>Example</em>:</p>
<div class="sourceCode" id="cb32"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb32-1"><a href="#cb32-1" aria-hidden="true"></a><span class="kw">using</span> graph <span class="op">=</span> gdwg<span class="op">::</span>graph<span class="op">&lt;</span><span class="dt">int</span>, <span class="dt">int</span><span class="op">&gt;</span>;</span>
<span id="cb32-2"><a href="#cb32-2" aria-hidden="true"></a><span class="kw">auto</span> <span class="kw">const</span> v <span class="op">=</span> std<span class="op">::</span>vector<span class="op">&lt;</span>graph<span class="op">::</span>value_type<span class="op">&gt;{</span></span>
<span id="cb32-3"><a href="#cb32-3" aria-hidden="true"></a>  <span class="op">{</span><span class="dv">4</span>, <span class="dv">1</span>, <span class="op">-</span><span class="dv">4</span><span class="op">}</span>,</span>
<span id="cb32-4"><a href="#cb32-4" aria-hidden="true"></a>  <span class="op">{</span><span class="dv">3</span>, <span class="dv">2</span>, <span class="dv">2</span><span class="op">}</span>,</span>
<span id="cb32-5"><a href="#cb32-5" aria-hidden="true"></a>  <span class="op">{</span><span class="dv">2</span>, <span class="dv">4</span>, <span class="dv">2</span><span class="op">}</span>,</span>
<span id="cb32-6"><a href="#cb32-6" aria-hidden="true"></a>  <span class="op">{</span><span class="dv">2</span>, <span class="dv">1</span>, <span class="dv">1</span><span class="op">}</span>,</span>
<span id="cb32-7"><a href="#cb32-7" aria-hidden="true"></a>  <span class="op">{</span><span class="dv">6</span>, <span class="dv">2</span>, <span class="dv">5</span><span class="op">}</span>,</span>
<span id="cb32-8"><a href="#cb32-8" aria-hidden="true"></a>  <span class="op">{</span><span class="dv">6</span>, <span class="dv">3</span>, <span class="dv">10</span><span class="op">}</span>,</span>
<span id="cb32-9"><a href="#cb32-9" aria-hidden="true"></a>  <span class="op">{</span><span class="dv">1</span>, <span class="dv">5</span>, <span class="op">-</span><span class="dv">1</span><span class="op">}</span>,</span>
<span id="cb32-10"><a href="#cb32-10" aria-hidden="true"></a>  <span class="op">{</span><span class="dv">3</span>, <span class="dv">6</span>, <span class="op">-</span><span class="dv">8</span><span class="op">}</span>,</span>
<span id="cb32-11"><a href="#cb32-11" aria-hidden="true"></a>  <span class="op">{</span><span class="dv">4</span>, <span class="dv">5</span>, <span class="dv">3</span><span class="op">}</span>,</span>
<span id="cb32-12"><a href="#cb32-12" aria-hidden="true"></a>  <span class="op">{</span><span class="dv">5</span>, <span class="dv">2</span>, <span class="dv">7</span><span class="op">}</span>,</span>
<span id="cb32-13"><a href="#cb32-13" aria-hidden="true"></a><span class="op">}</span>;</span>
<span id="cb32-14"><a href="#cb32-14" aria-hidden="true"></a></span>
<span id="cb32-15"><a href="#cb32-15" aria-hidden="true"></a><span class="kw">auto</span> g <span class="op">=</span> graph<span class="op">(</span>v<span class="op">.</span>begin<span class="op">()</span>, v<span class="op">.</span>end<span class="op">())</span>;</span>
<span id="cb32-16"><a href="#cb32-16" aria-hidden="true"></a>g<span class="op">.</span>insert_node<span class="op">(</span><span class="dv">64</span><span class="op">)</span>;</span>
<span id="cb32-17"><a href="#cb32-17" aria-hidden="true"></a><span class="kw">auto</span> out <span class="op">=</span> std<span class="op">::</span>ostringstream<span class="op">{}</span>;</span>
<span id="cb32-18"><a href="#cb32-18" aria-hidden="true"></a>out <span class="op">&lt;&lt;</span> g;</span>
<span id="cb32-19"><a href="#cb32-19" aria-hidden="true"></a><span class="kw">auto</span> <span class="kw">const</span> expected_output <span class="op">=</span> std<span class="op">::</span>string_view<span class="op">(</span><span class="st">R&quot;(1 (</span></span>
<span id="cb32-20"><a href="#cb32-20" aria-hidden="true"></a><span class="st">  5 | -1</span></span>
<span id="cb32-21"><a href="#cb32-21" aria-hidden="true"></a><span class="st">)</span></span>
<span id="cb32-22"><a href="#cb32-22" aria-hidden="true"></a><span class="st">2 (</span></span>
<span id="cb32-23"><a href="#cb32-23" aria-hidden="true"></a><span class="st">  1 | 1</span></span>
<span id="cb32-24"><a href="#cb32-24" aria-hidden="true"></a><span class="st">  4 | 2</span></span>
<span id="cb32-25"><a href="#cb32-25" aria-hidden="true"></a><span class="st">)</span></span>
<span id="cb32-26"><a href="#cb32-26" aria-hidden="true"></a><span class="st">3 (</span></span>
<span id="cb32-27"><a href="#cb32-27" aria-hidden="true"></a><span class="st">  2 | 2</span></span>
<span id="cb32-28"><a href="#cb32-28" aria-hidden="true"></a><span class="st">  6 | -8</span></span>
<span id="cb32-29"><a href="#cb32-29" aria-hidden="true"></a><span class="st">)</span></span>
<span id="cb32-30"><a href="#cb32-30" aria-hidden="true"></a><span class="st">4 (</span></span>
<span id="cb32-31"><a href="#cb32-31" aria-hidden="true"></a><span class="st">  1 | -4</span></span>
<span id="cb32-32"><a href="#cb32-32" aria-hidden="true"></a><span class="st">  5 | 3</span></span>
<span id="cb32-33"><a href="#cb32-33" aria-hidden="true"></a><span class="st">)</span></span>
<span id="cb32-34"><a href="#cb32-34" aria-hidden="true"></a><span class="st">5 (</span></span>
<span id="cb32-35"><a href="#cb32-35" aria-hidden="true"></a><span class="st">  2 | 7</span></span>
<span id="cb32-36"><a href="#cb32-36" aria-hidden="true"></a><span class="st">)</span></span>
<span id="cb32-37"><a href="#cb32-37" aria-hidden="true"></a><span class="st">6 (</span></span>
<span id="cb32-38"><a href="#cb32-38" aria-hidden="true"></a><span class="st">  2 | 5</span></span>
<span id="cb32-39"><a href="#cb32-39" aria-hidden="true"></a><span class="st">  3 | 10</span></span>
<span id="cb32-40"><a href="#cb32-40" aria-hidden="true"></a><span class="st">)</span></span>
<span id="cb32-41"><a href="#cb32-41" aria-hidden="true"></a><span class="st">64 (</span></span>
<span id="cb32-42"><a href="#cb32-42" aria-hidden="true"></a><span class="st">)</span></span>
<span id="cb32-43"><a href="#cb32-43" aria-hidden="true"></a><span class="st">)&quot;</span><span class="op">)</span>;</span>
<span id="cb32-44"><a href="#cb32-44" aria-hidden="true"></a>CHECK<span class="op">(</span>out<span class="op">.</span>str<span class="op">()</span> <span class="op">==</span> expected_output<span class="op">)</span>;</span></code></pre></div>
<p>—<em>end example</em> ]</p>
<h2 data-number="2.8" id="iterator-gdwg.iterator"><span class="header-section-number">2.8</span> Iterator [gdwg.iterator]<a href="#iterator-gdwg.iterator" class="self-link"></a></h2>
<div class="sourceCode" id="cb33"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb33-1"><a href="#cb33-1" aria-hidden="true"></a><span class="kw">template</span><span class="op">&lt;</span>concepts<span class="op">::</span>regular N, concepts<span class="op">::</span>regular E<span class="op">&gt;</span></span>
<span id="cb33-2"><a href="#cb33-2" aria-hidden="true"></a><span class="kw">requires</span> concepts<span class="op">::</span>totally_ordered<span class="op">&lt;</span>N<span class="op">&gt;</span> and concepts::totally_ordered&lt;E&gt;</span>
<span id="cb33-3"><a href="#cb33-3" aria-hidden="true"></a><span class="kw">class</span> graph<span class="op">&lt;</span>N, E<span class="op">&gt;::</span>iterator <span class="op">{</span></span>
<span id="cb33-4"><a href="#cb33-4" aria-hidden="true"></a><span class="kw">public</span><span class="op">:</span></span>
<span id="cb33-5"><a href="#cb33-5" aria-hidden="true"></a>  <span class="kw">using</span> value_type <span class="op">=</span> ranges<span class="op">::</span>common_tuple<span class="op">&lt;</span>N, N, E<span class="op">&gt;</span>;</span>
<span id="cb33-6"><a href="#cb33-6" aria-hidden="true"></a>  <span class="kw">using</span> difference_type <span class="op">=</span> std<span class="op">::</span><span class="dt">ptrdiff_t</span>;</span>
<span id="cb33-7"><a href="#cb33-7" aria-hidden="true"></a>  <span class="kw">using</span> iterator_category <span class="op">=</span> std<span class="op">::</span>bidirectional_iterator_tag;</span>
<span id="cb33-8"><a href="#cb33-8" aria-hidden="true"></a></span>
<span id="cb33-9"><a href="#cb33-9" aria-hidden="true"></a>  <span class="co">// Iterator constructor</span></span>
<span id="cb33-10"><a href="#cb33-10" aria-hidden="true"></a>  iterator<span class="op">()</span> <span class="op">=</span> <span class="cf">default</span>;</span>
<span id="cb33-11"><a href="#cb33-11" aria-hidden="true"></a></span>
<span id="cb33-12"><a href="#cb33-12" aria-hidden="true"></a>  <span class="co">// Iterator source</span></span>
<span id="cb33-13"><a href="#cb33-13" aria-hidden="true"></a>  <span class="kw">auto</span> <span class="kw">operator</span><span class="op">*()</span> <span class="op">-&gt;</span> ranges<span class="op">::</span>common_tuple<span class="op">&lt;</span>N <span class="kw">const</span><span class="op">&amp;</span>, N <span class="kw">const</span><span class="op">&amp;</span>, E <span class="kw">const</span><span class="op">&amp;&gt;</span>;</span>
<span id="cb33-14"><a href="#cb33-14" aria-hidden="true"></a></span>
<span id="cb33-15"><a href="#cb33-15" aria-hidden="true"></a>  <span class="co">// Iterator traversal</span></span>
<span id="cb33-16"><a href="#cb33-16" aria-hidden="true"></a>  <span class="kw">auto</span> <span class="kw">operator</span><span class="op">++()</span> <span class="op">-&gt;</span> iterator<span class="op">&amp;</span>;</span>
<span id="cb33-17"><a href="#cb33-17" aria-hidden="true"></a>  <span class="kw">auto</span> <span class="kw">operator</span><span class="op">++(</span><span class="dt">int</span><span class="op">)</span> <span class="op">-&gt;</span> iterator;</span>
<span id="cb33-18"><a href="#cb33-18" aria-hidden="true"></a>  <span class="kw">auto</span> <span class="kw">operator</span><span class="op">--()</span> <span class="op">-&gt;</span> iterator<span class="op">&amp;</span>;</span>
<span id="cb33-19"><a href="#cb33-19" aria-hidden="true"></a>  <span class="kw">auto</span> <span class="kw">operator</span><span class="op">--(</span><span class="dt">int</span><span class="op">)</span> <span class="op">-&gt;</span> iterator;</span>
<span id="cb33-20"><a href="#cb33-20" aria-hidden="true"></a></span>
<span id="cb33-21"><a href="#cb33-21" aria-hidden="true"></a>  <span class="co">// Iterator comparison</span></span>
<span id="cb33-22"><a href="#cb33-22" aria-hidden="true"></a>  <span class="kw">auto</span> <span class="kw">operator</span><span class="op">==(</span>iterator <span class="kw">const</span><span class="op">&amp;</span> other<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span>
<span id="cb33-23"><a href="#cb33-23" aria-hidden="true"></a><span class="kw">private</span><span class="op">:</span></span>
<span id="cb33-24"><a href="#cb33-24" aria-hidden="true"></a>  <span class="kw">explicit</span> iterator<span class="op">(</span><em>unspecified</em><span class="op">)</span>;</span>
<span id="cb33-25"><a href="#cb33-25" aria-hidden="true"></a><span class="op">}</span>;</span></code></pre></div>
<ol type="1">
<li><p>Elements are lexicographically ordered by their source node, destination node, and edge weight, in ascending order.</p></li>
<li><p>Nodes without any connections are not traversed.</p></li>
<li><p>[<em>Note</em>: <code class="sourceCode default">gdwg::graph&lt;N, E&gt;::iterator</code> models <a href="https://en.cppreference.com/w/cpp/iterator/bidirectional_iterator"><code class="sourceCode default">ranges::bidirectional_iterator</code></a>. —<em>end note</em>]</p></li>
</ol>
<h3 data-number="2.8.1" id="iterator-constructor-gdwg.iterator.ctor"><span class="header-section-number">2.8.1</span> Iterator constructor [gdwg.iterator.ctor]<a href="#iterator-constructor-gdwg.iterator.ctor" class="self-link"></a></h3>
<div class="sourceCode" id="cb34"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb34-1"><a href="#cb34-1" aria-hidden="true"></a>iterator<span class="op">()</span>;</span></code></pre></div>
<ol type="1">
<li><p><em>Effects</em>: Value-initialises all members.</p></li>
<li><p><em>Remarks</em>: Pursuant to the requirements of <a href="https://en.cppreference.com/w/cpp/iterator/forward_iterator"><code class="sourceCode default">std::forward_iterator</code></a>, two value-initialised iterators shall compare equal.</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb35"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb35-1"><a href="#cb35-1" aria-hidden="true"></a><span class="kw">explicit</span> iterator<span class="op">(</span><em>unspecified</em><span class="op">)</span>;</span></code></pre></div>
<ol start="3" type="1">
<li><p><em>Effects</em>: Constructs an iterator to a specific element in the graph.</p></li>
<li><p><em>Remarks</em>: There may be multiple constructors with a non-zero number of parameters.</p></li>
</ol>
<h3 data-number="2.8.2" id="iterator-source-gdwg.iterator.source"><span class="header-section-number">2.8.2</span> Iterator source [gdwg.iterator.source]<a href="#iterator-source-gdwg.iterator.source" class="self-link"></a></h3>
<div class="sourceCode" id="cb36"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb36-1"><a href="#cb36-1" aria-hidden="true"></a><span class="kw">auto</span> <span class="kw">operator</span><span class="op">*()</span> <span class="op">-&gt;</span> ranges<span class="op">::</span>common_tuple<span class="op">&lt;</span>N <span class="kw">const</span><span class="op">&amp;</span>, N <span class="kw">const</span><span class="op">&amp;</span>, E <span class="kw">const</span><span class="op">&amp;&gt;</span>;</span></code></pre></div>
<ol type="1">
<li><em>Effects</em>: Equivalent to:</li>
</ol>
<div class="sourceCode" id="cb37"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb37-1"><a href="#cb37-1" aria-hidden="true"></a><span class="kw">using</span> result_type <span class="op">=</span> ranges<span class="op">::</span>common_tuple<span class="op">&lt;</span>N <span class="kw">const</span><span class="op">&amp;</span>, N <span class="kw">const</span><span class="op">&amp;</span>, E <span class="kw">const</span><span class="op">&amp;&gt;</span>;</span>
<span id="cb37-2"><a href="#cb37-2" aria-hidden="true"></a><span class="cf">return</span> result_type<span class="op">(</span>pointee_<span class="op">-&gt;</span>from, pointee_<span class="op">-&gt;</span>to, pointee_<span class="op">-&gt;</span>weight<span class="op">)</span>;</span></code></pre></div>
<ol start="2" type="1">
<li><p><em>Remarks</em>: <code class="sourceCode default">ranges::common_tuple</code> is almost completely interface-compatible with <a href="https://en.cppreference.com/w/cpp/utility/tuple"><code class="sourceCode default">std::tuple</code></a>, but has subtle differences that make this assignment possible.</p></li>
<li><p>[<em>Note</em>: Do <strong>not</strong> use <code class="sourceCode default">ranges::make_common_tuple</code> or <code class="sourceCode default">std::make_tuple</code>, as these will always copy values instead of returning references. —<em>end note</em>]</p></li>
</ol>
<h3 data-number="2.8.3" id="iterator-traversal-gdwg.iterator.traversal"><span class="header-section-number">2.8.3</span> Iterator traversal [gdwg.iterator.traversal]<a href="#iterator-traversal-gdwg.iterator.traversal" class="self-link"></a></h3>
<div class="sourceCode" id="cb38"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb38-1"><a href="#cb38-1" aria-hidden="true"></a><span class="kw">auto</span> <span class="kw">operator</span><span class="op">++()</span> <span class="op">-&gt;</span> iterator<span class="op">&amp;</span>;</span></code></pre></div>
<ol type="1">
<li><em>Effects</em>: Advances <code class="sourceCode default">*this</code> to the next element in the range.</li>
</ol>
<p>[<em>Example</em>: In this way, your iterator will iterator through a graph like the one below producing the following tuple values when deferenced each time:</p>
<div class="sourceCode" id="cb39"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb39-1"><a href="#cb39-1" aria-hidden="true"></a>gdwg<span class="op">::</span>graph<span class="op">&lt;</span><span class="dt">int</span>, <span class="dt">int</span><span class="op">&gt;</span></span></code></pre></div>
<p><img src="https://qph.fs.quoracdn.net/main-qimg-2ea8bf9286505bf2ccd63893e05eb5f9" /></p>
<div class="sourceCode" id="cb1"><pre class="sourceCode txt"><code class="sourceCode default"><span id="cb1-1"><a href="#cb1-1" aria-hidden="true"></a>(1, 7, 4)</span>
<span id="cb1-2"><a href="#cb1-2" aria-hidden="true"></a>(1, 12, 3)</span>
<span id="cb1-3"><a href="#cb1-3" aria-hidden="true"></a>(1, 21, 12)</span>
<span id="cb1-4"><a href="#cb1-4" aria-hidden="true"></a>(7, 21, 13)</span>
<span id="cb1-5"><a href="#cb1-5" aria-hidden="true"></a>(12, 19, 16)</span>
<span id="cb1-6"><a href="#cb1-6" aria-hidden="true"></a>(14, 14, 0)</span>
<span id="cb1-7"><a href="#cb1-7" aria-hidden="true"></a>(19, 1, 3)</span>
<span id="cb1-8"><a href="#cb1-8" aria-hidden="true"></a>(19, 21, 2)</span>
<span id="cb1-9"><a href="#cb1-9" aria-hidden="true"></a>(21, 14, 23)</span>
<span id="cb1-10"><a href="#cb1-10" aria-hidden="true"></a>(21, 31, 14)</span></code></pre></div>
<p>—<em>end example</em>]</p>
<ol start="2" type="1">
<li><em>Returns</em>: <code class="sourceCode default">*this</code>.</li>
</ol>
<div class="sourceCode" id="cb40"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb40-1"><a href="#cb40-1" aria-hidden="true"></a><span class="kw">auto</span> <span class="kw">operator</span><span class="op">++(</span><span class="dt">int</span><span class="op">)</span> <span class="op">-&gt;</span> iterator;</span></code></pre></div>
<ol start="3" type="1">
<li><em>Effects</em>: Equivalent to:</li>
</ol>
<div class="sourceCode" id="cb41"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb41-1"><a href="#cb41-1" aria-hidden="true"></a><span class="kw">auto</span> temp <span class="op">=</span> <span class="op">*</span><span class="kw">this</span>;</span>
<span id="cb41-2"><a href="#cb41-2" aria-hidden="true"></a><span class="op">++*</span><span class="kw">this</span>;</span>
<span id="cb41-3"><a href="#cb41-3" aria-hidden="true"></a><span class="cf">return</span> temp;</span></code></pre></div>
<p><br /></p>
<div class="sourceCode" id="cb42"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb42-1"><a href="#cb42-1" aria-hidden="true"></a><span class="kw">auto</span> <span class="kw">operator</span><span class="op">--()</span> <span class="op">-&gt;</span> iterator<span class="op">&amp;</span>;</span></code></pre></div>
<ol start="4" type="1">
<li><p><em>Effects</em>: Advances <code class="sourceCode default">*this</code> to the previous element in the range.</p></li>
<li><p><em>Returns</em>: <code class="sourceCode default">*this</code>.</p></li>
</ol>
<p><br /></p>
<div class="sourceCode" id="cb43"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb43-1"><a href="#cb43-1" aria-hidden="true"></a><span class="kw">auto</span> <span class="kw">operator</span><span class="op">--(</span><span class="dt">int</span><span class="op">)</span> <span class="op">-&gt;</span> iterator;</span></code></pre></div>
<ol start="6" type="1">
<li><em>Effects</em>: Equivalent to:</li>
</ol>
<div class="sourceCode" id="cb44"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb44-1"><a href="#cb44-1" aria-hidden="true"></a><span class="kw">auto</span> temp <span class="op">=</span> <span class="op">*</span><span class="kw">this</span>;</span>
<span id="cb44-2"><a href="#cb44-2" aria-hidden="true"></a><span class="op">--*</span><span class="kw">this</span>;</span>
<span id="cb44-3"><a href="#cb44-3" aria-hidden="true"></a><span class="cf">return</span> temp;</span></code></pre></div>
<h3 data-number="2.8.4" id="iterator-comparison-gdwg.iterator.comparison"><span class="header-section-number">2.8.4</span> Iterator comparison [gdwg.iterator.comparison]<a href="#iterator-comparison-gdwg.iterator.comparison" class="self-link"></a></h3>
<div class="sourceCode" id="cb45"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb45-1"><a href="#cb45-1" aria-hidden="true"></a><span class="kw">auto</span> <span class="kw">operator</span><span class="op">==(</span>iterator <span class="kw">const</span><span class="op">&amp;</span> other<span class="op">)</span> <span class="op">-&gt;</span> <span class="dt">bool</span>;</span></code></pre></div>
<ol type="1">
<li><em>Returns</em>: <code class="sourceCode default">true</code> if <code class="sourceCode default">*this</code> and <code class="sourceCode default">other</code> are pointing to elements in the same range, and <code class="sourceCode default">false</code> otherwise.</li>
</ol>
<h2 data-number="2.9" id="compulsory-internal-representation-gdwg.internal"><span class="header-section-number">2.9</span> Compulsory internal representation [gdwg.internal]<a href="#compulsory-internal-representation-gdwg.internal" class="self-link"></a></h2>
<p>Your graph is <strong>required</strong> to own the resources (nodes and edge weights) that are passed in through the insert functions. This means creating memory on the heap and doing a proper copy of the values from the caller. This is because resources in your graph should outlive the caller’s resouce that was passed in in case it goes out of scope. For example, we want the following code to be valid.</p>
<div class="sourceCode" id="cb46"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb46-1"><a href="#cb46-1" aria-hidden="true"></a><span class="kw">auto</span> main<span class="op">()</span> <span class="op">-&gt;</span> <span class="dt">int</span> <span class="op">{</span></span>
<span id="cb46-2"><a href="#cb46-2" aria-hidden="true"></a>  gdwg<span class="op">::</span>graph<span class="op">&lt;</span>std<span class="op">::</span>string, <span class="dt">int</span><span class="op">&gt;</span> g;</span>
<span id="cb46-3"><a href="#cb46-3" aria-hidden="true"></a>  <span class="op">{</span></span>
<span id="cb46-4"><a href="#cb46-4" aria-hidden="true"></a>    std<span class="op">::</span>string s1<span class="op">{</span><span class="st">&quot;Hello&quot;</span><span class="op">}</span>;</span>
<span id="cb46-5"><a href="#cb46-5" aria-hidden="true"></a>    g<span class="op">.</span>insert_node<span class="op">(</span>s1<span class="op">)</span>;</span>
<span id="cb46-6"><a href="#cb46-6" aria-hidden="true"></a>  <span class="op">}</span></span>
<span id="cb46-7"><a href="#cb46-7" aria-hidden="true"></a></span>
<span id="cb46-8"><a href="#cb46-8" aria-hidden="true"></a>  <span class="co">// Even though s1 has gone out of scope, g has its own</span></span>
<span id="cb46-9"><a href="#cb46-9" aria-hidden="true"></a>  <span class="co">//  copied resource that it has stored, so the node</span></span>
<span id="cb46-10"><a href="#cb46-10" aria-hidden="true"></a>  <span class="co">//  will still be in here.</span></span>
<span id="cb46-11"><a href="#cb46-11" aria-hidden="true"></a>  std<span class="op">::</span>cout <span class="op">&lt;&lt;</span> g<span class="op">.</span>is_node<span class="op">(</span><span class="st">&quot;Hello&quot;</span><span class="op">)</span> <span class="op">&lt;&lt;</span> <span class="st">&quot;</span><span class="sc">\n</span><span class="st">&quot;</span>; <span class="co">// prints &#39;true&#39;;</span></span>
<span id="cb46-12"><a href="#cb46-12" aria-hidden="true"></a><span class="op">}</span></span></code></pre></div>
<p>Your graph is <strong>required</strong> to use smart pointers (however you please) to solve this problem.</p>
<ol type="1">
<li><p>For each edge, you are only allowed to have one underlying resource (heap) stored in your graph for it. [<em>Note</em>: You may store a unique weight multiple times, but no more than once for each distinct edge with that weight.—<em>end note</em>]</p></li>
<li><p>For each node, you are only allowed to have one underlying resource (heap) stored in your graph for it.</p></li>
<li><p>[<em>Hint</em>: In your own implementation you’re likely to use some containers to store things, and depending on your implementation choice, somewhere in those containers you’ll likely use either <code class="sourceCode default">std::unique_ptr&lt;N&gt;</code> or <code class="sourceCode default">std::shared_ptr&lt;N&gt;</code> —<em>end hint</em>]</p></li>
</ol>
<h3 data-number="2.9.1" id="but-why-smart-pointers-gdwg.internal.rationale"><span class="header-section-number">2.9.1</span> But why smart pointers [gdwg.internal.rationale]<a href="#but-why-smart-pointers-gdwg.internal.rationale" class="self-link"></a></h3>
<p>You could feasibly implement the assignment without any smart pointers, through lots of redundant copying. For example, having a massive data structure like:</p>
<div class="sourceCode" id="cb47"><pre class="sourceCode cpp"><code class="sourceCode cpp"><span id="cb47-1"><a href="#cb47-1" aria-hidden="true"></a>std<span class="op">::</span>map<span class="op">&lt;</span>N, std<span class="op">::</span>vector<span class="op">&lt;</span>std<span class="op">::</span>pair<span class="op">&lt;</span>N, E<span class="op">&gt;&gt;&gt;</span></span></code></pre></div>
<p>You can see that in this structure you would have duplicates of nodes when trying to represent this complex structure. This takes up a lot of space. We want you to build a space efficient graph. This means only storing one instance of each node and edge.</p>
<h2 data-number="2.10" id="other-notes-other.notes"><span class="header-section-number">2.10</span> Other notes [other.notes]<a href="#other-notes-other.notes" class="self-link"></a></h2>
<p>You must:</p>
<ul>
<li>Include a header guard in <code class="sourceCode default">include/gdwg/graph.hpp</code></li>
<li>Use C++20 style and methods where appropriate</li>
<li>Make sure that <em>all appropriate member functions</em> are <code class="sourceCode default">const</code>-qualified</li>
<li>Leave a moved-from object in a state with no nodes.</li>
<li>Implement this class within the namespace <code class="sourceCode default">gdwg</code>.</li>
<li>Assignment 2 asked you to implement <code class="sourceCode default">operator!=</code> because you’ll see it in a lot of production codebases, and it’s important that you know how to write it correctly. To get used to how C++20 handles <code class="sourceCode default">operator!=</code>, we’re asking that you <strong>do not</strong> implement an overload for <code class="sourceCode default">operator!=</code> in Assignment 3.</li>
<li><strong>Write a <a href="https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet">Markdown</a> document <code class="sourceCode default">test/README.md</code> explaining your rationale behind the decisions you made while testing.</strong></li>
</ul>
<p>You must not:</p>
<ul>
<li>Write to any files that aren’t provided in the repo (e.g. storing your vector data in an auxilliary file)</li>
<li>Add additional members to the <b style="color: red;">public</b> interface.</li>
</ul>
<p>You:</p>
<ul>
<li>Should try to mark member functions that will not throw exceptions with <code class="sourceCode default">noexcept</code></li>
<li>Are not required to make any member function explicit unless directly asked to in the spec.</li>
</ul>
<h3 data-number="2.10.1" id="const-correctness-const.correctness"><span class="header-section-number">2.10.1</span> <code class="sourceCode default">const</code>-correctness [const.correctness]<a href="#const-correctness-const.correctness" class="self-link"></a></h3>
<p>We have deliberately removed the <code class="sourceCode default">const</code>-qualifiers for most member functions from the specification. <strong>You are required to work out which functions must be <code class="sourceCode default">const</code>-qualified.</strong> You must ensure that each operator and member function appropriately either has:</p>
<ul>
<li>A <code class="sourceCode default">const</code> member function; or</li>
<li>A non-<code class="sourceCode default">const</code> member function; or</li>
<li>Both a <code class="sourceCode default">const</code> and non-const member function</li>
</ul>
<p>Please think carefully about this. The function declarations intentionally do not specify their constness, except for the <code class="sourceCode default">begin()</code> and <code class="sourceCode default">end()</code> member functions. These are <code class="sourceCode default">const</code>-qualified to help you out.</p>
<p>In most cases you will only need a single function in the overload set.</p>
<h1 data-number="3" id="getting-started"><span class="header-section-number">3</span> Getting Started<a href="#getting-started" class="self-link"></a></h1>
<p>If you haven’t done so already, clone this repository.</p>
<div class="sourceCode" id="cb2"><pre class="sourceCode sh"><code class="sourceCode bash"><span id="cb2-1"><a href="#cb2-1" aria-hidden="true"></a>$ <span class="fu">git</span> clone gitlab@gitlab.cse.unsw.edu.au:z5555555/20T2-cs6771-ass3</span></code></pre></div>
<p>(Note: Replace z5555555 with your zid)</p>
<p>Navigate inside the directory. You can then open vscode with <code class="sourceCode default">code .</code> (note the dot).</p>
<h2 data-number="3.1" id="running-your-tests"><span class="header-section-number">3.1</span> Running your tests<a href="#running-your-tests" class="self-link"></a></h2>
<p>Similar to the first tutorial, you simply to have to run <code class="sourceCode default">Ctrl+Shift+P</code> and then type <code class="sourceCode default">Run Test</code> and hit enter. VS Code will compile and run all of your tests and produce an output.</p>
<h2 data-number="3.2" id="adding-more-tests"><span class="header-section-number">3.2</span> Adding more tests<a href="#adding-more-tests" class="self-link"></a></h2>
<p>Part of your assignment mark will come from the quality and extensiveness of tests that you write.</p>
<p>You can add more test files to the <code class="sourceCode default">test/graph/</code> directory. Simply copy <code class="sourceCode default">test/graph/graph_test1.cpp</code> into another file in that directory.</p>
<p>Note, everytime that you add a new file to the <code class="sourceCode default">test/graph/</code> directory you will need to add another few lines to <code class="sourceCode default">test/CMakeLists.txt</code>. You can once again, simply copy the test reference for <code class="sourceCode default">graph_test1.cpp</code> and rename the appropriate parts. Every time you update <code class="sourceCode default">CMakeLists.txt</code> in any repository, in VSCode you should codess <code class="sourceCode default">Ctrl+Shift+P</code> and run <code class="sourceCode default">Reload Window</code> for the changes to take effect.</p>
<h1 data-number="4" id="marking-criteria"><span class="header-section-number">4</span> Marking Criteria<a href="#marking-criteria" class="self-link"></a></h1>
<p>This assignment will contribute 20% to your final mark.</p>
<p>The assessment for the assignment recognizes the difficulty of the task, the importance of style, and the importance of appropriate use of programming methods (e.g. using while loops instead of a dozen if statements).</p>
<table class="table table-bordered table-striped">
<tr>
<td align="right">
50%
</td>
<td>
<b>Correctness</b><br /> The correctness of your program will be determined automatically by tests that we will run against your program. You will not know the full sample of tests used prior to marking.
</td>
</tr>
<tr>
<td align="right">
25%
</td>
<td>
<b>Your tests</b><br /> You are required to write your own tests to ensure your program works. You will write tests in the <code>test/</code> directory. <b>Write a <a href="https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet">Markdown</a> document <code class="sourceCode default">test/README.md</code> explaining your rationale behind the decisions you made while testing.</b> Please read the <a href="https://github.com/catchorg/Catch2/blob/master/docs/tutorial.md">Catch2 tutorial</a> or review lecture/tutorial content to see how to write tests. Tests will be marked on several factors. These include, but are not limited to:
<ul>
<li>
Correctness — an incorrect test is worse than useless.
</li>
<li>
Coverage - your tests might be great, but if they don’t cover the part that ends up failing, they weren’t much good to you.
</li>
<li>
Brittleness — If you change your implementation, will the tests need to be changed (this generally means avoiding calling functions specific to your implementation where possible - ones that would be private if you were doing OOP).
</li>
<li>
Clarity — If your test case failed, it should be immediately obvious what went wrong (this means splitting it up into appropriately sized sub-tests, amongst other things).
</li>
</ul>
</td>
</tr>
<tr>
<td align="right">
20%
</td>
<td>
<b>C++ best practices</b><br /> Your adherence to good C++ best practice in lecture. This is <b>not</b> saying that if you conform to the style guide you will receive full marks for this section. This 20% is also based on how well you use modern C++ methodologies taught in this course as opposed to using backwards-compatible C methods. Examples include: Not using primitive arrays and not using pointers. We will also penalise you for standard poor practices in programming, such as having too many nested loops, poor variable naming, etc.
</td>
</tr>
<tr>
<td align="right">
5%
<td>
<b>clang-format</b><br /> In your project folder, run the following commands on all cpp/h files in the <code class="sourceCode default">source</code> and <code class="sourceCode default">test</code> directory.<br /> <code>$ clang-format-11 -style=file -i /path/to/file.cpp</code> <br /> We will run <code>clang-format-11 -style=file /path/to/file.cpp | diff /path/to/file.cpp -</code> on all your headers and source files. If, for each of these files, <code>diff</code> outputs nothing (i.e. is formatted correctly), you will receive full marks for this section (5%). A video explaining how to use clang-format can be found <a href="https://unsw.zoom.us/rec/share/4_1tE-mo2UpJRo2V0U_UVp4BRoDAeaa8hicbrvVfxRs2wxZeMb3Au5H-75jpZZWj">HERE</a>.
</tr>
</table>
<p>The following actions will result in a 0/100 mark for this assignment, and in some cases a 0 for COMP6771:</p>
<ul>
<li>Knowingly providing your work to anyone and it is subsequently submitted (by anyone).</li>
<li>Submitting any other person’s work. This includes joint work.</li>
</ul>
<p>The lecturer may vary the assessment scheme after inspecting the assignment submissions but it will remain broadly similar to the description above.</p>
<h1 data-number="5" id="originality-of-work"><span class="header-section-number">5</span> Originality of Work<a href="#originality-of-work" class="self-link"></a></h1>
<p>The work you submit must be your own work. Submission of work partially or completely derived from any other person or jointly written with any other person is not permitted.</p>
<p>The penalties for such an offence may include negative marks, automatic failure of the course and possibly other academic discipline. Assignment submissions will be examined both automatically and manually for such submissions.</p>
<p>Relevant scholarship authorities will be informed if students holding scholarships are involved in an incident of plagiarism or other misconduct.</p>
<p>Do not provide or show your assignment work to any other person — apart from the teaching staff of COMP6771.</p>
<p>If you knowingly provide or show your assignment work to another person for any reason, and work derived from it is submitted, you may be penalized, even if the work was submitted without your knowledge or consent. This may apply even if your work is submitted by a third party unknown to you.</p>
<p>Note you will not be penalized if your work has the potential to be taken without your consent or knowledge.</p>
<h1 data-number="6" id="submission"><span class="header-section-number">6</span> Submission<a href="#submission" class="self-link"></a></h1>
<p>This assignment is due <em>Monday 3rd of August, 19:59:59</em>. Submit the assignment using the following comand while logged into the CSE machines:</p>
<div class="sourceCode" id="cb3"><pre class="sourceCode sh"><code class="sourceCode bash"><span id="cb3-1"><a href="#cb3-1" aria-hidden="true"></a><span class="ex">6771</span> submit ass3</span></code></pre></div>
<p>This will submit whatever is on the master branch of THIS repository (the one this README.md file is contained in) at the moment of submission.</p>
<p>Please ensure that you can build and run your tests successfully on the CSE machine.</p>
<h1 data-number="7" id="late-submission-policy"><span class="header-section-number">7</span> Late Submission Policy<a href="#late-submission-policy" class="self-link"></a></h1>
<p>If your assignment is submitted after this date, each hour it is late reduces the maximum mark it can achieve by 2%.</p>
<p>For example if an assignment you submitted with a raw awarded mark of 85% was submitted 5 hours late, the late submission would have no effect (as maximum mark would be 90%).</p>
<p>If the same assignment was submitted 20 hours late it would be awarded 60%, the maximum mark it can achieve at that time.</p>
<h1 data-number="8" id="cmake-tools-workaround"><span class="header-section-number">8</span> CMake Tools workaround<a href="#cmake-tools-workaround" class="self-link"></a></h1>
<p>There appears to be a bug in the configuration stage for the CMake Tools VS Code extension, when you already have an existing build cache (build directory). It’s unlikely that this bug will be fixed before the term ends, so we’ve updated the project settings for VS Code so that CMake Tools won’t auto-configure for you any more, but this means that you’ll manually need to run this <em>very specific</em> configuration step.</p>
<p>Instead of running <code class="sourceCode default">CMake: Configure</code> when you press <em>Ctrl+Shift+P</em>, you’ll need to run <code class="sourceCode default">CMake: Delete Cache and Reconfigure</code>. This will delete your build cache and require you to rebuild everything, but at least it will work without major confusion.</p>
<p><strong>The critical part to remember is that unlike in Assignments 1 and 2 where you could add a new file and it would automatically reconfigure, you will need to re-run this step every single time you edit any <code class="sourceCode default">CMakeLists.txt</code>.</strong></p>
</div>
</div>
</body>
</html>
