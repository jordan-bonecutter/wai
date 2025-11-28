# Proposal for the wai programming language
Programming languages having been reaching ever nearer and nearer to wai. New
programming languages tend to look like either Rust or Zig. Rust is very strict
and very safe, mostly because it does a lot of work to prove that your code is
safe. Once you want to interface with foreign code, implement an interpreter,
or do something significantly low level it requires a good amount of unsafe
code.

Languages like Zig take a different approach, but don't give you nearly as much
help to prove that your code is safe. The type system is also nowhere near as
good as Rust's typesystem which is unmatched (no pun intended) in my opinion.

So what do we do? Are we forced to choose between safety and dynamics? I think
not. The core of the issue is that the power has not been given to the 
programmer which is always a mistake. Compilers too smart, programmers too 
powerless.

Wai is a programming language which looks a lot like Rust in terms of its type
system but more like Zig and C when it comes to dealing with pointer safety. 
When you look closer you'll see that it's something new entirely.

The name is chosen after the letter "Y" as it's the second to last letter in
the english alphabet. Y will be the first version of this language which I know
will be full of poor design decisions. Z will be its successor, and the last
compiled programming language ever to be designed (I jest).

Enough talk though, here is some wai code:

```rust
// wai has a similar concept to Rust enums called patterns. In fact, for all
// intents and purposes patterns are just enums. With 1 important difference
// though, you can specify an implementation of how the pattern will be stored
// in memory. If you don't specify an implementation then wai will auto 
// implement the pattern for you exactly as a Rust enum would be laid out in
// memory using a raw union and a variant field. As such this is a contrived 
// example but useful nonetheless.
pattern Option<T> {
	None,
	Some(T),
}

// OptionImpl is a struct which we'll use to implement the option pattern. It
// has a where clause which the compiler uses to prove that the code is valid.
// Where clauses are the real innovation of the wai programming language. Where
// clauses can state facts about your code which will cause compilation errors
// if the compiler cannot prove them true. Here we have created a where clause
// which states an iff condition and creates a relationship between the ok field
// of an OptionImpl and a data field. Namely this iff condition states that iff
// the ok field is true then we can be certain that the data contained in the
// data field is valid. This is probably a good point to talk about data 
// validity. Maybe it's simplest to say when data isn't valid:
// 1. uninitialized variables is invalid
// 2. data that's been moved out is invalid
//      - moving data happens when its ownership moves to another piece of code
//      - you Rust fans will be glad to know, wai's ownership model is very
//        similar to Rust's!
// Any other data is considered to be valid. A nice thing to know about invalid
// data is that we don't need to drop it.
//
// More to be written soon on what can be done with invalid state. We'll see an
// example of it in the OptionImpl Option implementation.
// 
// Wai also has raw unions. Raw unions can often lead to compiler errors merely
// by constructing one because the compiler doesn't know which drop 
// implementation to call. Here we've fully specified to the compiler how to
// tell if data is valid so the compiler won't complain when we create one and
// can automatically implement drop.
struct OptionImpl<T> {
	data: (invalid | valid)T,
	ok: bool,
} where ok <=> valid(data);

// Here we implement the Option pattern using the OptionImpl struct.
impl<T> Option<T> using OptionImpl<T> {
    // set the data in the OptionImpl to correspond to an Option.
    // The pointer to self here is a write pointer, which means it can only
    // write to the data it points at. This allows us to pass in a pointer 
    // which points at invalid data and is very useful for move elision (imagine
    // a very large object, you may want to declare one separately from 
    // initializing it. Before it's been initialized it is invalid. Initializers
    // can be written by taking write pointers to self).
	fn set(
		&write self,
		variant: Option::Variant,
		data: T | () 
	) where (
        variant == Option::Variant::None <=> typeof(data) == ()
    ) && (
        variant == Option::Variant::Some <=> typeof(data) == T
    )
    // This where clause sets up an iff condition between the variant value and
    // the data value. This helps us figure out which type data is, since it's
    // a raw union.
    {
		match(variant) {
			Option::Variant::None: {
				*self = Self{ok: false};
			},
			Option::Variant::Some: {
				*self = Self{ok: true, data};
			}
		}
	} where valid(*after(self));
    // here we have a final where clause which states something about the 
    // validity of the self pointer. In order to deal with mutability wai has
    // 2 special functions whcih can be used in where clauses: before and after.
    // this statement says that after the function is called we can guarantee
    // that the data pointed at by self is valid. This lets us use this function
    // as an initializer so that any code which uses this can be certain that
    // the type is initialized.

    // get returns the data contained in self. Read pointers are essentially
    // shared references in Rust or const references in C++.
    // here we also have a where clause after we declare the return value.
    // as of this proposal I haven't decided where the where clauses should
    // go syntactically, but I think all of these places are valid.
	fn get(&read self) -> (
		variant: Option::Variant,
		data: &T | &()
	) where (
        variant == Option::Variant::None <=> typeof(data) == ()
    ) && (
        variant == Option::Variant::Some <=> typeof(data) == T
    ) {
		if self.ok {
			return (Option::Variant::None, &self.data);
		}

		return (Option::Variant::Some, &self.data);
	}
}
```

The next example shows what interfacing with some 3rd party code might look
like. Often interfacing with 3rd party code involves using antipatterns which
should be discouraged but most certainly supported!

```rust
// KernelHandle is a type which gets responses from the kernel and lets you read
// them into a kernel response. It's left blank as the actual data is not very
// important here.
struct KernelHandle {}

// KernelResponse is the data that the kernel responds with.
// For this request the kernel either responds with an int or a float and tells
// us which one it is given the first value, resp_type.
struct KernelResponse {
    resp_type: isize,
    resp: i32 | f32
} where resp_type == 0 <=> typeof(resp) == i32, resp_type == 1 <=> typeof(resp) == f32;

impl KernelHandle {
    // This function is does something a little weird:
    // - Take a reference to potentially uninitialized data
    // - If a response is ready then write the response into that reference
    // - Otherwise write nothing
    // - Return a bool to the caller which is true iff the data is now valid.
    //
	// This function implements somewhat of an antipattern as you should just
	// use an option, but sometimes you don't have the luxury of doing so (we
	// may be interfacing with foreign code or implementing something very low 
	// level).
	fn read(&read self, result: &write KernelResponse) -> (ok bool) {
		if self.ready() {
            // trustme is the "unsafe" of wai. It tells the compiler that it
            // cannot prove the statement inside the brackets but it should
            // assume that the where clause following is true. Generally where
            // clauses may follow any statement which may help the compiler 
            // prove the truth of your code and prevent bugs.
            //
            // The only times you need to use trustme are:
            // 1. Interfacing with foreign code.
            // 2. Doing something intentionally unsafe. We'll go over such an
            //    example later.
            trustme {
                C::read_response_into(result);
            } where valid(*after(result));
			return true;
		}

		return false;
	} where ok => valid(*after(result));
}

// Here is another example of a valid use of trustme. Wai will never let you
// misinterpret data and always assumes it's a mistake. But sometimes you want
// to intentionally misinterpret data.
fn print_f32_bits(data: f32) {
    let bits = trustme{
        data
    } where typeof(data) == i32;
	print(`${bits.bitstring()}`);
}

```

The final example is a sketch implementation of vec. I'm sure there are many
things wrong with it but it gets the point across and also introduces the 
concept of planes which is very important to how Wai proves memory safety.

```rust
struct Vec<T, A=std::DefaultAllocator> {
	len: isize,
	cap: isize,
	data: plane[isize][T],
    alloc: A,
} where len <= cap;
// a plane is an owned piece of memory not created by the compiler. memory created
// by the compiler is stack and global memory. Whenever a method returns a reference
// Wai must determine which plane that reference lives on. This helps Wai prove
// memory disjointness and prevent dangling pointers.

impl Vec<T, A> {
	fn new() -> (ret Self) {
		Self{len: 0, cap: 0, data: plane::empty(), alloc: A::new()}
	}

	fn length(&read self) -> (length: isize) {
		return self.len;
	} where self.len == length;
    // The above where clause helps us safely index into the array at compile
    // time.
	
	fn index(&read self, index: isize) -> &T {
		return &self.data[index];
	} where index >= 0 && index < self.len;
    // this function can only be called if you can prove the index is between
    // 0 and self.len.

    fn needs_realloc(&read self) -> bool {
        self.len == self.cap
    }

	fn try_push(&self) -> Result<(), A::Error> {
		if self.needs_realloc() {
			self.try_realloc()?;
		}

		self.data[self.len] = value;
	} where 
        (before(self.needs_realloc()) || default) => invalid(before(self.data));
    // this where clause is how we prevent against the classic following case:
    /*
        let v = Vec<i32>::new();
        v.push(10);
        let ref_0 = &v[0];
        v.push(11);
        print(`${*ref_0}`);
    */
    // because push may reallocate we need to tell Wai that the plane before 
    // running this function has been made invalid. If the caller of this 
    // function knows if it's going to need_realloc they can actually continue
    // using pointers dealt out from it which is why before(self.needs_realloc())
    // is the left side of the implication. If we don't know (meaning the caller
    // of this function didn't bother to check) then we always assume that
    // the data is invalid which is accomplished by || default.
    // try_push also uses the error bubble up operator: ?.

    // push is the same as try_push but uses the error assertion operator which
    // stops the program if an error is encountered.
	fn push(&self, value: T) {
		self.try_push()!;
	} where (before(self.needs_realloc()) || default) => invalid(before(self.data));

    // try_index is like index but doesn't assume you know anything about len.
	fn try_index(&read self, index: isize) -> (ret Option<&T>) {
		if index < self.len && index >= 0 {
			return Some(self.index(index));
		}

		return None;
	}

    fn drop(&self) {
        if self.cap > 0 {
            self.a.drop(self.plane)
        }
    }
}
```
