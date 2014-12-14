ARM vs x86: Pathfinding benchmark of Rust, G, Haskell, OCaml, Nimrod, F#, Dart, Racket, Common Lisp, and C++

In this benchmark I thought it would be interesting to explore a less common pathfinding algorithm. Imagine you've just been "invited" to visit your in-laws (if you don't have inlaws, imagine you're driving a married friend), and want to find the longest possible route to the in-laws' house, in order to minimise the time spent with them. Now, you don't want your spouse (or your friend's spouse) to know you're stalling, so you can't visit the same place twice, can't just spend infinity hours driving in a circle. How would you find this longest path?

One way is to create a graph with the nodes representing different intersections and the connections between these nodes representing the distance of the roads between these intersections. One can then solve it with the relatively simple approach of iterating through all possible routes to find the longest. You may be thinking this sounds incredibly slow, and it is, something like O(n!). Unfortunately however there are no known "fast" algorithms that can find this path; the problem is considered NP complete, meaning if you can solve it in O(n^b) time, where b is a constant, then you get a [million dollar prize](http://en.wikipedia.org/wiki/Millennium_Prize_Problems#P_versus_NP).




Go

Not really much to say here. The implementation was pretty simple to write, and achieved reasonable performance without any effort put into optimising the code. One thing to note is that it was the only language apart from Rust that required me to explicitly handle the failure case of atoi when parsing in the routes, albeit in a somewhat ugly manner:

    node, err1 := strconv.Atoi(nums[0])
    neighbour, err2 := strconv.Atoi(nums[1])
    cost, err3 := strconv.Atoi(nums[2])
    if err1 != nil || err2 != nil || err3 != nil{
      panic("Error: encountered a line that wasn't three integers")
    }

It actually enforced it less strongly than Rust, as technically I could have just ignored the errors returned by assigning them to _, but doing so would generally be considered an abomination in good Go code.

Rust

This was by far the hardest implementation to write, due to the language having changed significantly since last I used it. Relatively simple taskes eluded me, for instance:

let mut visited: Vec<bool> = Vec::from_fn(10, |_| false);
visited[2] = true;

This looks like it should create a mutable vector of 10 bools set to false, and then set the one at index 2 to true. Does it compile? No, it prints:

error: cannot assign to immutable dereference (dereference is implicit, due to indexing)

Okay, maybe I need to make a mutable dereference of visited[2]:

let mut thirdBool = &(visited[2]);
*thirdBool= false;

This should work, right? Nope, turns out I `cannot assign to immutable dereference of `&`-pointer `*thirdBool``. Hmm, so I need a mutable dereference, whatever that is, but I've only got a pointer. Okay, how about:

let mut thirdBool = &mut (visited[2]);

That looks like it should give me an mutable reference to visited[2]. Does it? No, because I `cannot borrow immutable dereference (dereference is implicit, due to indexing) as mutable`. :'( I also checked the Vec docs and the introduction to Rust, but couldn't find any mention of how to set a vector element.

Needless to say, being the lazy sod that I am, I turned to a simpler solution.

  unsafe{
    let newAddr: uint = (visited as uint) + (2 as uint);
    let newAddrP: *mut bool = newAddr as *mut bool;
    *newAddrP = true;
  }

Problem solvered! This led me to the somewhat frightening discovery that unsafe is not transitive: when I put the above code in the findPlaces function, that function doesn't need to be marked as unsafe and the code calling it doesn't need to be in an unsafe block. This is probably necessary, in the sense that the standard library uses unsafe code for performance and it wouldn't make sense for all stdlib calls to be marked unsafe, but I nevertheless find it surprising how easy it is. Maybe it could be useful to require a compiler flag for compiling code containing `unsafe`, like C# does.

That being said, the version I was using is the latest available in the Arch Linux repositories, 0.12, and it's quite possible that the newest version makes mutating an element in a vector much easier.

I found it slightly disappointing that Rust has chosen to forbid top-level function type inference, of the kind possible in F#, OCaml and Haskell. The argument is that this prevents the interface-breaking that could occur if a change within an exported function altered its signature, but this wouldn't be a problem with a proper module system like OCaml's, in which the module interface must be specified explicitly and if function signatures within the module don't match this then there's a compilation error. Imagine a function that takes a channel of mutable references to atomically reference counted options of ints: in Rust I'd have to write something like:

Chan<&mut Vec<RC<RefCell<Option<int>>>>

in the function type signature, which is rather gangly, especially if the signature also needed to include some lifetimes. In OCaml (or Haskell), in contrast, I could just write the function, press a button to have Emacs automatically generate the signature, then copy the signature to the module file (or place it above the function definition, in Haskell's case). An ML style module system would also bring Rust higher kinded types for free (OCaml's higher kinded polymorphism functionality relies on its module system). Even Haskell is now moving towards a ML module system, via Backpack.

D

Like Go, writing the D implementation was pretty straightforward. I did however make one mistake that manifested in a rather hilariously unrelated error: I declared the array of nodes with

node[] nodes =  uninitializedArray!(node[])(numNodes);

Then, when I attempted to append a new neighbour to one of the nodes on the list, the program failed with an out of memory exception. Why? Turned out, the unitialised array of nodes wasn't zeroed, so the length and capacity parts of each node's neighbour array were presumably full of gibberish, causing the append function to attempt to allocate an absurd amount of memory to append to them. Changing `unitializedArray!` to `minimallyInitializedArray!` fixed this.

Racket

The Racket implementation was the first Lisp implementation I wrote, as the use of Typed Racket made it easier to write, with the type checker catching most of the many errors I made before runtime. I only found one instance of the type system slowing me down: trying to convert a string to an integer. The code I wound up with was:

(: str->int (String -> Integer))
(define (str->int str) 
  (define n (string->number str))
  (if n (numerator (inexact->exact (real-part n))) 0))

First, it calls string->number, which returns a (U Integer nil), a union type. The `if n` is like a form of pattern matching, converting n from type (U Integer nil) to just Integer, which seems pretty neat. The verbose and potentially unnecessary part is `(numerator (inexact->exact (real-part n)))`, which first takes the real part of n (it could be a constant), then converts it to exact (it could be a float), then takes the numerator (it could be a fraction), to finally give an int. A str->int function in the stdlib that returned (U Integer nil) could be a nice alternative to this.

Common Lisp

This was a bit more tricky to write than the Racket implementation. Common Lisp doesn't come with a string split algorithm in its standard library, so I borrowed one I found online. It also doesn't allow recursive function definitions in let bindings, instead requiring the use of `labels`. The biggest inconvenience I found, however, was the apparently lack of the equivalent of Racket's "build vector", which builds a vector from a function. Common Lisp has `make-array`, but if that is used with a struct initial element then it just fills every element of the vector with a reference to the same struct instance, causing a modification of one element to effect the rest. I hence had to populate the vector manually:

(dotimes (i num-nodes)
	(setf (elt nodes i) (make-node)))

Which admittedly isn't much code to write.

I did however find Common Lisp's type system far more useful in terms of obtaining the performance benefits of types. Unlike Typed Racket, Common Lisp uses gradual typing, so you can add type declarations for one function or variable without having to add them to others. I also found developing Common Lisp with Slime and Emacs a lot smoother than developing Racket on DrRacket, largely because SBCL compiles code far quicker.

OCaml

Ocaml was a delight to write as usual, thanks partially to the fantastic OCaml emacs plugins, Tuareg and Merlin. They provide error checking upon saving, like Eclipse with Java, and easy code testing and reloading via their integration with the OCaml repl. While not as neat as the Haskell implementation, the OCaml implementation was quicker to write, due to not having to deal with monadic IO, and performs impressively fast. Interestingly, I found my initial imperative version, which used an ioref, was actually slightly slower than the functional version, possibly due to the GC's write barrier.

FSharp

It was pretty simple to translate the OCaml implementation into F#, apart from a few minor differences like `array.(myIndex)` changing to `array.[myIndex]`. The F# was however nowhere near as fast as the OCaml. Interestingly, the F# Emacs support was even better, with fsharp-mode enabling some kind of extremely powerful intellisense, although the use of intellisense required the creation of an xml-laden myfile.fsproj file for some reason.

Java

Really not much to say here. Verbose, but fairly simple to write, and reasonably fast, although not comparable to the compiled languages.

Haskell

The Haskell implementation was generally pleasant to write; the Vector.modify function proved to be extremely convenient for building the node vector. It takes a vector-mutating function and returns either a copy of the vector with that function applied or the same vector mutated by that function, depending on whether or not it is safe to do the latter. This allows a vector to be mutated in pure code, via the ST monad, which is much quicker than having to allocate a new vector.

Interestingly, when I was attempting to modify the code to be more functional (passing max along in a fold rather than mutating it as an ioref), I realised I didn't understand do notation as well as I thought I did.

Can you spot the mistake in the following code? I didn't.

do
  UMV.write visited nodeID True
  let max = GV.foldM' acc 0 (nodes V.! nodeID)
  UMV.write visited nodeID False
  return max

This lead to the program using all the memory and dying, leading me to think there was a memory leak in acc, although I checked it thoroughly and couldn't find one. Turns out, the above code is actually the equivalent of

do
  UMV.write visited nodeID True
  UMV.write visited nodeID False
  return $ GV.foldM' acc 0 (nodes V.! nodeID)

To fix it, I needed to change

let max = GV.foldM' acc 0 (nodes V.! nodeID)

to:

max <- GV.foldM' acc 0 (nodes V.! nodeID)

One thing I found less pleasant than in OCaml was the autoindentation support. This is not the fault of the plugin itself, but rather the fact that indentation in Haskell affects meaning: whenever the semantics of code could depend on the indentation level, the autoindenter doesn't have one true indentation to select as the indentation depends on what you want the code to do. In a language without significant indentation, like a Lisp or a curly braces language, 'one true indentation' is possible.


Dart

Similar to Java, the Dart implementation was generally quite simple to write... apart from the Async nature of IO, which initially caught me off guard.

readPlacesAndFindPath() {
  var nodes;
  new File('agraph').readAsLines().then((List<String> lines) {
    final int numNodes = int.parse(lines[0]);
    final nodes = new List<Node>.generate(numNodes, (int index) => new Node()); 
    for(int i = 1; i < lines.length; i++){
      final nums = lines[i].split(' ');
      int node = int.parse(nums[0]);
      int neighbour = int.parse(nums[1]);      
      int cost = int.parse(nums[2]);
      nodes[node].neighbours.add(new Route(neighbour,cost));
    }
  });
  return nodes;
}

What does the above do? Answer: it returns null, as the .then is asynchronous, meaning the function will not wait for nodes to be populared before returning. A stronger type system (or a read of the dart::io documentation) would have caught this.