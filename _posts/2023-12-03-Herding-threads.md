---
layout: post
title: "Herding Threads: Taming Data Races with Parametrized Algebraic Effects"
date: 2023-12-03
---
### Anurag Choubey
---

Full bibliographic citation for research paper: Hashan Punchihewa and Nicolas Wu. 2021. Safe mutation with algebraic effects. In Proceedings of the 14th ACM SIGPLAN International Symposium on Haskell (Haskell 2021). Association for Computing Machinery, New York, NY, USA, 122–135. https://doi.org/10.1145/3471874.3472988

---

## Introduction
Concurrent programming is the art of writing programs where multiple threads run at the same time. They may also share the same resources for read/write operations. A simple example may be a concurrent increment of every element in the array, where each thread has a reference to the array and performs the increment at each location (ideally) independently. Concurrent programs often lead to a boost in the performance of the program, but one of their undesirable effects is the occurrance of data races.
<br>
Let us understand it more formally. A state associated with an entity is mutable if it can be changed after the entity is created.
Data races occur when, in the absence of a synchronization mechanism,  multiple threads attempt to simultaneously access the state of an entity, with at least one of them attempting an update. Consider the example below:
<br>
<br>
```haskell
sitOnChair member chairId isFree = do
    occupation ← read isFree
    if not occupation
        then do 
            write occupation True
            putStrLn (member ++ " sat on chair " ++ show chairId ++ " when the music stopped!")
    else 
        putStrLn ("Sorry " ++ member ++ " someone already got to your chair, you're out!")

racyMusicalChair = do
    chairsLeft ← alloc 1  
    david ← alloc "David"
    peter ← alloc "Peter"
    fork (sitOnChair david 5 False) 
    fork (sitOnChair peter 5 False)
```
<br>
In the above example, the following data race condtions could occur:<br>

- Both Sit Down: Due to the race condition, it's possible that both players' actions read isFree as False before either has a chance to set it to True. This would lead to both players believing they have successfully sat down, which is not possible in the actual game.
- Both Out: If the operations are interleaved in a specific manner, both players might end up getting the message that the chair is already taken.
<br>
Mainstream approaches to tackle such issues are to use software transactional memory, where mutations to a shared state performed by multiple threads, are combined and then updated all at once (atomically) or implementing the actor model, where there is no shared state and actors coordinate by passing immutable data. But both the approches have their shortcomings. While the software transactional memory model relies on a runtime system to manage the programs interaction with memory, the use of the actor model requires restructuring the code with respect to the actors and their coordination primitives, which introduces bugs that are a class of their own. The goal of the paper mentioned above is to study a third approach, ie, compile time verification of absence of data race conditions. This is acheived via tracking the use of mutable states by incorporation of advanced type systems. 
<br>
The paper starts out by first discussing the idea of algebraic effects and how they may be encoded in a language such as Haskell. Building on that discussion , the paper constructs a programming model that utilizes parametrized algebraic effects and type-level stat tracking to disallow data races at compile time. While there is also a type checker plugin discussed in the paper, the authors felt it was beyond the scope of this post and hence it has been omitted. Curious readers may find it in section 5 of the paper.

---

## 1. On Algebraic Effects

### 1.1 Algebraic Effects 101
Algebraic effects are a method in functional programming to handle operations like reading a file or updating a variable, which are called side effects. They separate what the effects do from how they are implemented, much like defining an interface. This allows you to write code that's easy to understand and change, because the side effects are managed separately from your main logic. They help keep functions pure, meaning the same inputs always give the same outputs, which is a key principle in functional programming. Algebraic effects make your code more flexible, letting you easily swap out different ways of handling side effects without changing the rest of your code.
The paper encodes algebraic effects using a functor. The operations that the functor performs are encoded as data constructors of that functor.
<br>As an example, consider the code snippet below 
```haskell
data Store a where
Alloc :: a → Store (Token a) 
Read :: Token a → Store a 
Write :: Token a → a → Store ()
```
The Store functor "stores" values of type "a". Alloc allocates a new mutable store with an initial value. Note that Alloc creates a Store functor that contains mutable references to values of type a, and not actual values of type a. The mutable references are of the type Token Read reads a value from the store. Read :: Token a -> Store a takes a token and returns the value in the store. Write updates the value in the store. Write :: Token a -> a -> Store () takes a token and a new value to update the store. Now that we have our starting piece, we need some way to do computations with our representatio of an algebraic effect. We may do this by combining the Store functor with the Prog monad.
```haskell
data Prog f a where
Return :: a → Prog f a
Call ::fb→(b→Progfa)→Progfa
```
Prog has two constructors:
Return: Used for pure computations.
Call: Used for the storage actions (like putting in or taking out values of type a). We may understand how Prog may be used along with Store to implement a mutable store effect
```haskell
read s = Call (Read s return) 
write s v = Call (Write s v return)
------ ------ ------
increment s = do
v ← read s 
write s (v + 1)
```
read and write are smart constructors defined to acheive the intended behavior. Let us take a look at write

```haskell
write s v = Call (Write s v return)
{-
s is of type Token a.
v is of type a.
Write s v then is of type Store () because Write :: Token a → a → Store ().

The Call constructor requires:

Its first argument to be of type f b for some effect f and type b. 
In this case, f is Store, and b is (). So, Write s v fits as it is of type Store ().
Its second argument to be a function of type b → Prog f a. Since b is (), the function is () → Prog f a.

The return function in Haskell can serve this purpose as it takes
any value (including ()) and lifts it into a monad, in this case, Prog f.
Thus, Call (Write s v return) represents a Prog computation where an effectful
operation (Write) is performed, and then the result (which is of type ()) is passed to return, 
which does nothing significant here but is necessary to fit the Call constructor's type requirements.
-}
```
One may understand the read constructor in a similar manner. We may use these smart constructors to write a
meaningful operation as follows:
```haskell
increment s = do
v ← read s 
write s (v + 1)
```
Overall, using a Store functor and combining it with the Prog monad gives us a powerful mechanism for writing programs that can express mutable store effects.

#### SMALL problem

The above abstraction cannot be used to write constructs such as fork. Fork is an operation that spawns a child thread. The reason for this is because of the commutativity requirement
for an algebraic effect to work with monadic bind. Simply put, the order of  applying monadic operations and bind must not change the final effect acheived. Doing an operation and then binding should be the same as binding and then doing the operation.<br>
The key problem is that fork does not commute with bind. The equivalence (fork p) >>= k is not the same as fork (p >>= k).
- (fork p) >>= k: Here, you first fork the process p, creating a new thread, and then apply the continuation k to the result of fork in the original thread.
- fork (p >>= k): In this case, you're saying to first bind p with k, creating a new combined operation, and then fork this entire new operation.

In the first case we are implying that operation k affects the parent thread only, whereas in the second thread the operation k shall affect the parent as well as the child thread. The two cases shall have different effects produced after the computations are carried out, which breaks the commutativity. This shortcoming of algebraic effects can be addressed via an enhancement known as scoped effects.

### 1.2 Scoped Algebraic Effects

The paper describes a scoped operation as "an operation which changes the behaviour of subsequent operations." For any given scoped operation , the set of operations affected by the given scoped operation is known as an inner scope. In Haskell, the scope defined by an operation must be made explicit via syntax. Let us understand it with the help of some code.

```haskell
data Prog f g a where
Return :: a → Prog f g a
Call :: fb → (b → Prog f g a) → Prog f g a
Enter :: Prog f g a → g a b → (b → Prog f g c) → Prog f g c
```
We can begin understanding how the above modification to the Prog monad incorporates scoped operations by carefully looking at the Enter constructor.
- Here Prog f g a is the inner scope. In terms of concurrency, it is the "section" of the computation being performed by a child thread.
- g a b is the scoped operation. It's like saying, “Here’s an operation that takes the context of type a, does something scoped (defined by g), and then produces a result of type b.” Note that g itself is an effect (similar to Store) and has data constructors of its own that define operations to it.
- b -> Prog f g c represents the continuation, it takes the result of applying the scoped computation to the inner scope and "exposes" it to the outer scope.
- Finally Prog f g c represents the outer scope. In terms of concurrency, it reperesents the section of the computation to be carried out by parent thread after the child thread has finished.

```haskell
data Thread a b where
     Fork :: Thread () ()
---------------------------------------
fork p = Enter Fork p return
```
In the code given above Fork is a scoped operation for which the Thread functor is used.<br>
The smart constructor fork p = Enter Fork p return is a way to define a scoped computation with effects. Here,
- p is of type Prog f g a , which is the operation that must be carried out by then child thread.
- Fork is of type Thread () () (think :: g a b). When fork p is executed, the Enter constructor initiates the scoped effect Fork, starting a new thread.
- The computation p is then executed in this new thread. This is where the actual "forking" happens - p runs independently of the main thread.
- return takes the result of the computation in the child thread and "lifts" it up to the scope of the parent thread.

#### OK, but how did it solve my commutativity problem?

By scoping the operations associated with an effect f , in Prog f g a, we bypass the need for commutativity. By adding a scoped ooeration g a b in the Enter constructor, what we are saying to Haskell is to not bind the result of Prog f g a → g a b in the ordinary chained manner as we are used to. Enter lets you apply a scoped effect (like fork) and clearly define where this effect starts and ends. It doesn't mix up with the regular bind sequence; instead, it creates a distinct context for the effect to operate. For now, think scoping as delineating boundaries for operations of effects.

## 2. On Type Level States

In the previous section an abstraction has been defined to handle effectful scoped computations. Alas, this syntax alone is not enough to prevent both David and Peter ending up on the same chair (remember David and Peter from Introduction?). In threaded computations , reading from a store is safe if and only if no other thread is attempting a simultaneous write, and a write operation is safe if and only if not other thread has access to the store at all. This is where the paper introduces the concept of access levels.
Safety of the read/write operations can be modelled using three access levels:
- N : A thread with access level N can neither read from nor write to a store
- R : A thread with access level R can only read from a store
- X : A thread with access level X can read and write to a store and it has exclusive access.

```haskell
data AccessLevel = N | R | X
{-
N, R and X are ordered as N <= R <= X
Below is a typeclass to capture this relationship
-}
type (⩽) :: AccessLevel → AccessLevel → Constraint classa⩽b
instance a ⩽ X
instance R ⩽ R
instance N ⩽ R instance N ⩽ N
```
Let us go back to the increment example in section 1.1. Let the access level of the store s (which has value of type a) be given by m.
The read operation will generate a constraint  R <= m , meaning that the thread doing the reading can atleast read the store. The write operation will generate the constraint X <= m. Everything happens without conflict in the above single-threaded scenario. In our model, however, threads WILL be created. Since our threads are scoped, an operation of the inner scope may change the access level to a resource. This may lead to a scenario where the child thread modifies the access level in such a way that the parent thread loses it.
<br>
In order to understand and work with the relationship between the access level of a parent thread before fork , access level of a parent thread after fork and access level of a child thread , we will introduce a new typeclass, as defined below

```haskell
type Share :: AccessLevel → AccessLevel → AccessLevel → Constraint
class Share a b c 
instance Share X X N 
instance Share X N X 
instance Share X R R 
instance Share R R R 
instance Share N N N
```

A Constraint is generated by taking as inputs the  access level of a parent thread before fork , access level of a parent thread after fork and access level of a child thread.

<!-- If your site is at https://username.github.io/concurrencyChronicles/ -->
<img src="/concurrencyChronicles/images/image.png" width="400" height="300" />



<br>Consider the example below:
```haskell
goodState = do 
 s ← alloc True
 write s False 
 fork do
    write s False
 return ()
```
Here the goodState program shall generate a state X X N, which means that the parent thread has write access before fork, the child thread has write access and the parent thread neither read not write access. This constraint can be satisfied because it is an instance of the Share typeclass. Consider another example.
```haskell
badState = do
s ← alloc True
fork do
    write s False 
write s False
return ()
```
The Constraint generated by badState is X X X, which cannot be satisfied. We can also intuitively understand why this is so. The parent thread has write access before fork, the child thread has write access and the parent thread has write access even after the fork. Such a constraint would obviously lead to data race conditions.

## Parametrised Algebraic Effects

In the previous sectio we saw how we can prevent data races by qualifying only those computations that satisfy a given Constraint. But how would we go about actually implementing such type-level state behavior in Haskell? This is where the notion of parametrised computation comes in. Parametrised computation is how we "restrict" a computation based on the state of an object that is changed by that computation. The paper dives into the more theoretical aspects of this concepts and the curious reader is encouraged to read it. The state of the object before the computation is the precondition and the state after is known as the postcondition. In Haskell such a concept may be encoded as
```haskell
type IFunctor :: (p → p → Type → Type) → Constraint 
class IFunctor f where
  imap :: (a → b) → f i j a → f i j b
```
- The parameters p represent state conditions (like preconditions and postconditions). The third parameter in f represents the data type the functor operates on.
- The imap function essentially says that if you have a way to transform a value of type a into type b, and you have a computation in f that produces a value of type a, you can transform this computation into one that produces a value of type b, without altering the underlying state transition.

Following the trend from the previous sections, since we have to enable support for effectful computations, we are required to make a monad class that is an instance of this functor.

```haskell
type IMonad :: (p → p → Type → Type) → Constraint 
class (IFunctor m) ⇒ IMonad m where
  ireturn :: a → m i i a
  ijoin ::m i j (m j k a) → m i k a
  ibind ::m i j a → (a → m j k b) → m i k b
```

Similar to IFunctor, IMonad is a type constructor that operates on a type m, which itself is a function of two state parameters and a type, producing a Type. The state parameters (p) represent the state transitions.
- ireturn :: a → m i i a : List a value of type a to a monadic computation, where there is no state transition (i i)
- ijoin ::m i j (m j k a) → m i k a : Handles the computations where the state transitions from i to j and then from j to k. Main purpose is to flatten a nested monadic structure.
- ibind :: m i j a → (a → m j k b) → m i k b: This is similar to the monadic operation bind (>>=). It is used for chaining parametrised computations together. 

#### Okay , but how does this fit in with effects?

So far within this section we have introduced an abstration for placing access "restrictions" by introducing a notion of precondition and postcondition which can then be used to "monitor" access safety. We have also provided a software defiition in terms of the IMonad typeclass. We ust somehow now also come up with a software abstraction that allows us to seamlessly chain together such parametrised computations but now also with the production AND handling of algebraic effects.<br>
To that end, the paper defines a parametrised monad IProg. Just like we defined the notion of an algebraic effect using the Prog monad earlier, we are going to use IProg to define a parametrised computation with algebraic effects.
```haskell
data IProg f p q a where
  Return :: a → IProg f p p a
  Call ::f p q b → (b → IProg f q r a) → IProg f p r a  
```
The execution flow is similar to the Prog monad defined earlier. We may also look at an example of how this may work with a made-up effect
```haskell
data Status = Active | Inactive
type Firewall :: Status → Status → Type → Type
data Firewall p q x where
  Enable :: Firewall Inactive Active () 
  Disable :: Firewall Active Inactive ()
```
Enable and Disable are operations on the effect Firewall. Take a look at Enable. In order for a call to Enable to typecheck it HAS TO HAVE as input an effect whose precondition is Inactive and the postcondition as Active, else it will throw an error.

#### A PitStop

- So far we have seen the notion of an algebraic effect, and we used them to model stores for values that may be mutated, and such mutations have an "effect" in the overall computation.
- Then we went on to solve the commutativity problem of forking a threaded computation that may  have an algebraic effect. For that we made our effect model ito a scoped one, with an inner scope, a scoped operation and an outer scope.
- Since the above two still did not solve how we could place access restirctions on resources after thread creation, we introduced the concept of access levels and constraints produced by forking operations which have to be satisfied at compile time in order for the computation to be valid.
- To model the idea of access levels and constraint satisfaction, we worked with parametrised algebraic effects in order to have effectful computations BUT which also respect a SPECIFIC pre-to-postcondition "mapping". This is acheived via the IProg data type.

Okay, we can drive off from our pitstop with the following question in mind.
We have parametrised our algebraic effects......but where did the whole idea of scoping vanish in the previous two sections? Last we checked, we also need to respect the commutativity of monadic binds, right?

```haskell
data IProg f g p q x where
Return :: a → IProg f p p a
Call :: f p q b → (b → IProg f q r a)→ IProg f p r a
Enter :: IProg f g p′ q′ a → g p p′ q′ q a b → (b → IProg f g q r c) → IProg f g p r c
```
Here the line of greatest importance is operstion Enter. Let us break it down:<br>
- Inner Scope = IProg f g p′ q′ a . Here p' and q' represent the pre and post condition of the inner scope computation. 
- g p p′ q′ q a b is a little complicated. p is the state before entering the inner scope, p' is the state after entering inner scope, q' is the state before exiting inner scope and q is the state after exiting the inner scope.
- IProg f g p r c is the outer scope computation.

Thus to conclude up till now we have discussed parametrised algebraic effects that also have support for scoped computations. 


## Type Level State Revisited

So far we have discussed parametrised effects in the abstract sense, in this section we shall introduce a version of store effect (discussed earlier) which is now parametrized. The parametrized state shall be represented by an array of AccessLevel (discussed earlier).
```haskell
type Token :: Type -> Nat -> Type
```
Token is used to represent a mutable data store in the program. Originally , Token was indexed by a single type variable, this time though, we shall introduce an addtion index of type Nat , which is the index to the corresponding access level in the parametrized state.
```haskell
type Store :: [AccessLevel] → [AccessLevel] → Type 
data Store p q x where
    Alloc :: t → Store p (Append p X) (Token t (Length p))
    Write :: (X ⩽ Lookup p n) ⇒ Token t n → t → Store p p ()
    Read :: (R ⩽ Lookup p n) ⇒ Token t n → Store p p t
```
Notice that by parametrizing the operations of the Store effect now with an array of access levels, and using indexed tokens to refer to mutable stores, we make it impossible to write programs that give rise to data races. And we do it at the type level!
<br>
Let us now introduce a parametrized functor that we can work with in terms of concurrency.
```haskell
type Thread :: [AccessLevel] → [AccessLevel] → [AccessLevel] → [AccessLevel] → Type → Type → Type
data Thread p p′ q′ q x x′ where
    Fork::(ShareList p q p′) ⇒ Thread p p′ q′ q () ()

type ShareList :: [AccessLevel] → [AccessLevel] → [AccessLevel] → Constraint
class ShareList as bs cs
instance ShareList [] [] []
instance (Share a b c, ShareList as bs cs) ⇒ ShareList (a : as) (b : bs) (c : cs)
```

- The four lists of AccessLevel in Thread functor represent the state of access levels before and after a scoped thread operation (like Fork).
- What is ShareList?? : Remember Share from when we first discussed type level states? That is how we were generating constraints on computations. The last instance declaration of ShareList typeclass that the heads satisfy the Share constraint as well as the tails satisfy the ShareList constraint.

Thus by defining the Thread functor, we have a parametrised algebraic effect that supports scoped operations as well as prevents data races by design.

### The problem of lost access

```haskell
lostAccess = do 
    store ← alloc True
    fork do
        write store False
        return () 
write store False
```

Such a program would generate the constraint Share X X X, which is unsatisfiable. The main reason for such a problem is that the parent thread has no idea if and when the child thread has finished. If we restrict ourselves to writing concurrent programs where the loss of a resource is assumed when a child thread accesses it, it would severly limit the range of programs we could write. To fix such a situation we introduce the fiish operation. What finish does is that it will wait for ALL the computations which are the descendants of the its inner scope computation. 
```haskell
data Thread p p′ q′ q x x′ where 
    ...
Finish :: Thread p p q p x x
```
Notice the scope before and after entering the forked op is p itself, meaning that each child thread spawned has now completed its operation. Such a construct now allows us to write something like..
```haskell
lostAccessRegained = do
     store ← alloc True 
     finish do
        fork (write store False) 
write store False
```

## Putting it all Together : The Array Functor

We now introduce an Array functor that allows us to allocate data in form of an Array and apply effectful operations to that Array. Owing to the heavy emphasis on technical details in the previous sections, we shall not dwelve into the implementtion of the functor. Instead we shall understand it through a simple example

```haskell
quicksort :: (X ⩽ Lookup p n)
             ⇒ Token Int r n
             → IProg Array Thread p p () 
quicksort arr = do
    len ← length arr 
    if len ⩽ 2 then do
        when (len ≡ 2) do
            v0 ← read arr 0
            v1 ← read arr 1
            when (v0 > v1) do
                write arr 0 v1
                write arr 1 v0 
    else finish do
        i ← partition len arr
        (i1, i2) ← slice arr (max (i − 1) 0)
        fork do
            quicksort i1
        quicksort i2
        return ()
```
- It takes a mutable reference to an array as a Token, and returns a parametrised monadic computation that handles effects.
- The interesting case is when the length of the array is > 2, then it uses the finish operation to perform the recursive case, this ensures that any operation after the quicksort has the same access to the array as before the quicksort.
- slice is a smart wrapper for an operation on the Array effect, which allocates meaningful chunks of the array for the forked threads to work on.
- Thus we see how any race conditions that could arise as a result of buggy code could be checked at compile time.

To conclude our summary, each section discussed basically works towards ensuring that the mutations caused to a value in multithreaded operations are safe in nature ad no data race conditions occur. The folowing section concludes the summary with a fun assignment idea!

