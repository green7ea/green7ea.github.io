# Modern C++

**Modern C++** embraces features from **C++11** which layed the
foundation for a new generation of C++. It introduced **move
semantics** and embraced **RAII in the standard library**
(`unique_ptr`, `shared_ptr`).

Embracing **R**esource **A**cquisition **I**s **I**nitialization
(RAII) makes C++ the safest, most productive and fun C++ has ever
been. Unfortunately, to leverage RAII, you need a good
**understanding** of **why it's needed** and **how it works** which is
what I hope to distill for you.

I'll explain RAII by example while referencing the [C++ core
guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#main).

# What is RAII?

The idea behind [RAII](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rr-raii) is to **manage resources using variables and
scope** so that resources are automatically freed when they are no
longer used. A resource is anything that can be acquired and must
later be released.

| Resource         |             |
|------------------|-------------|
| Memory           | new/delete  |
| File descriptors | open/close  |
| Lock             | lock/unlock |

The main idea is to acquire a **resource** which gives us a handle
that we **store in a `private` member** variable in a class. We can
then control its access and **release the resource in `destructor`**.

We'll create a **safe wrapper around a UNIX file descriptor** keeping
in mind that the same concept can be applied to any other
resource. Normally, the **standard library has a robust
implementation** already in place so make sure you check there first
to save yourself some work.

# Naive implementation

We'll start off with the **following definition** and assume that
there's a **reasonable implementation** for each prototype.

```c++
class NaiveFile {
public:
  NaiveFile(const std::string &path);
  ~NaiveFile();

  /// Reads up to 1024 characters. If
  /// something goes wrong, it returns
  /// an empty string.
  std::string read_1024() const;

private:
  int fd;
};
```

The following code would **behave in ways that are unintended**. We
can highlight it with the following usage:

```c++
// Notice we pass by copy
void accidental_copy(NaiveFile file);

int main() {
  NaiveFile naive_file = NaiveFile(filename);

  // Copy 1, since we pass by copy
  accidental_copy(naive_file);

  // Copy 2, we used the = operator that makes a copy
  auto file2 = naive_file;
}
```

**Running** the above code with **trace printing** results in the
following:

```
(fd 3) open main.cpp
(fd 3) ~NaiveFile closing
(fd 3) ~NaiveFile closing
  (fd 3) Couldn't close file: 'Bad file descriptor'
(fd 3) ~NaiveFile closing
  (fd 3) Couldn't close file: 'Bad file descriptor'
```

We can see that the **destructor is called 3 times** and we **free the
resource each time**. This happens once for the main instance and once
for each copy as the comments predicted. We **shouldn't free a
resource more than once**, so we get a 'Bad file descriptor' error
when we try to close the file the second or third time.

# Understanding the problem

Before we fix the problem, it's a good idea to **understand what's
going on**. The first thing to know is that, under certain conditions,
**C++ compilers implement a default** version of the following:

| Can be defaulted    |                                     |
|---------------------|-------------------------------------|
| default constructor | `NaiveFile()`                       |
| destructor          | `~NaiveFile()`                      |
| copy constructor    | `NaiveFile(const NaiveFile &)`      |
| copy assignment     | `void operator=(const NaiveFile &)` |
| move constructor    | `NaiveFile(NaiveFile &&)`           |
| move assignment     | `void operator=(NaiveFile &&)`      |

These are known as special operations and together they control the
life cycle (create, copy, move, destroy) of the object ([guidelines
C.ctor](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-ctor)). For RAII, we don't have to worry about the default
constructor since it's either a sane default or we implemented a
constructor ourselves.

The default implementations are pretty simple — if you have **no
implementation**, a **default** is put in place that **calls the same
operation for each member**. If a **member has no implementation**,
the **default isn't created**.

In our case, we implemented a constructor so there won't be a default
constructor. The only member is an `int`, so all the other defaults
will be created.

|                            | Compiler destructor | Compiler copy operations | Compiler move operations |
|----------------------------|---------------------|--------------------------|--------------------------|
| No user special operation  | defaulted           | defaulted                | defaulted                |
| User destructor exists     | —                   | **defaulted**            |                          |
| User copy operation exists | defaulted           | **defaulted**            | deleted                  |
| User move operation exists | defaulted           |                          |                          |

So the problem is that the **compiler implemented a default copy
constructor** (triggered by copy 1) and a **default copy assignment**
(triggered by copy 2). Those are highlighted in the table. The default
that is put in place treats the `int` as copyable. That's a problem
for us because **file descriptors aren't simple numbers** that can be
copied but represents a handle to a resource the kernel holds for us.

# The fix — rule of 3

The fix is pretty easy, **make copies impossible** by deleting the
operations that we don't want to default. With this change, the
**destructor will only be called once** and the **resource will only
be freed once**.

```c++
class SafeFile {
public:
  SafeFile(const std::string &path);
  SafeFile(const SafeFile &) = delete;
  ~SafeFile();
  void operator=(const SafeFile &) = delete;

  /// Reads up to 1024 characters. If
  /// something goes wrong, it returns
  /// an empty string.
  std::string read_1024() const;

private:
  int fd;
};
```

Now, our **file wrapper can't be misused** since trying to call a
deleted operation will cause a compilation error. It gets a promotion
to `SafeFile` from `NaiveFile`.

As a rule **if you implement a destructor or a copy operation**,
you're likely doing resource management and have to **implement all
3**. If you don't, you're likely to end up with an error like we did
above. This is known as **the rule of 3**.

| Rule of 3        |
|------------------|
| Destructor       |
| Copy constructor |
| Copy assignment  |

We see in the table above that if we `delete` the copy operations, the
**move operations also get deleted**. This makes our **wrapper less
useful** because the resource is stuck in the context where it was
created. Implementing a move lets us safely transfer it to a new
context.

In are case it doesn't matter but we are also potentially **giving up
performance** if we **implement the copy operations but not the
move**. If we have a value that could be moved (`rvalue`) but no move
operation is available, the value will be copied instead of moved
which can be costly.

Because of the two use cases above, you usually want to **expand the
rule of 3** to include the **move constructor** and **move
assignment** as well. This is known as [the rule of five](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-five). So the
**rule of 3 makes us safe** and the **rule of 5 gives performance and
convenience**.

| Rule of 5        |
|------------------|
| Destructor       |
| Copy constructor |
| Copy assignment  |
| Move constructor |
| Move assignment  |

I wondered why the rule of 3 and the rule of 5, **aren't enforced by
the compiler**. It **couldn't be an error** because that would break
too many existing programs but it **could be a warning**. I looked
into it and it turns out the C++ committee did exactly this in C++11
by updating the wording ([PDF, section 12.8/7](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3337.pdf)) for this behaviour
so that such a warning could be generated.

I had never seen this warning since GCC and Clang hide it behind the
`-Wdeprecated-copy-dtor` flag which is **disabled by default** and
**almost never enabled** (not part of `-Wall`, `-Wextra` or
`-Wpedantic`) — hopefully, I remember to turn it on for my future
projects.

# Nice wrapper — Rule of 5

To round up our example, let's **upgrade it** from using the rule of 3
**to the rule of 5** by implementing the **move operations**.

```c++
class FileWrapper {
public:
  FileWrapper(const std::string &path);
  FileWrapper(const FileWrapper &) = delete;
  FileWrapper(FileWrapper &&);
  ~FileWrapper();
  void operator=(const FileWrapper &) = delete;
  void operator=(FileWrapper &&);

  /// Reads up to 1024 characters. If
  /// something goes wrong, it returns
  /// an empty string.
  std::string read_1024() const;

private:
  int fd { -1 };
};
```

There's an important implementation detail with the **move
operations**, the **destructor** of the **object that was moved will
still be called** but it **shouldn't free the resource**. This means
that we have to put the original object in a state where the resource
won't be released by the destructor.

In our example, we do that by setting the original `fd` to `-1` as we
can see bellow. This works because an `fd` smaller than `0` indicates
that the open operation wasn't successful and shouldn't be closed.

```c++
FileWrapper::~FileWrapper() {
  if (fd >= 0) close(fd);
}

FileWrapper::FileWrapper(FileWrapper &&wrapper)
: fd(wrapper.fd) {
  // 'wrapper' won't close a fd with -1 in its
  // destructor.
  wrapper.fd = -1;
}

// We do something similar for operator=
```

To sum up the latest changes:

| Method        | Description                                            |
|---------------|--------------------------------------------------------|
| `constructor` | opens a file, `fd` >= 0 when successful                |
| `move`        | takes `fd` from the original and sets original to `-1` |
| `destructor`  | frees the resource if opened successfully (>= 0)       |

# Implications — rule of zero

We should now have a good enough understanding of RAII to put it to
use. Once we have a RAII wrapper around a resource, it becomes very
hard to misuse it which allows us to create **sealed abstractions
around resources**.

One important detail that briefly mentioned is that a `struct` or
`class` that uses our `FileWrapper` won't have default copy or move
operations. This is because `FileWrapper` doesn't have those
operations so they can no longer be defaulted. The implication of this
is huge, **any class that properly implements RAII can safely be put
in any other class** and it will just work.

We've successfully **isolated all the complexity of resource
management** into our wrapper class. This makes **modern C++** feel
**like a garbage collected language** only it's **faster** and it
**works for any resource**, not just memory. This happens because the
**defaults just do the right thing**.

We can see bellow that resources can be used as part of other
structures or as part of a scope and it's safe and just works.

```c++
struct SafeStuct {
  std::string name;
  FileWrapper fd;
};

function scoped_resource() {
    // Do work that doesn't need a lock

    // Scope to control lock duration
    {
      std::lock_guard<std::mutex> guard(mutex);
      // Work that needs a lock
    }

    // Do more work that doesn't need a lock
}
```

In a codebase, there are usually have a **few resource types** that
are **used all over the code**. For **each resource**, we use an
**RAII wrapper** from the standard library or in a rare case, we
implement one ourselves — this is the only time we really need to
think about resource management. In the **rest of the code**, which is
the **majority**, we **don't worry about resource management**. C++'s
defaults of the special operations do the heavy lifting for us, this
is known as the [rule of zero](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-zero). This frees us from having to do
resource management which is what makes modern C++ so pleasant.

# Limitations

RAII makes resource management extremely safe, even when exceptions
cause an unpredicted control flow to happen, but it has one main
limitation — **RAII doesn't prevent use after free**. If you move a
resource, you can still use it afterwards which is a bug.

# Further reading

- [foonathan, When to Write Which Special Member](https://www.foonathan.net/2019/02/special-member-functions/),
- [MC++, C++ Core Guidelines: The Rule of Zero, Five, or Six](https://www.modernescpp.com/index.php/c-core-guidelines-constructors-assignments-and-desctructors/),
- [Bjarne Stroustrup, Writting Good C++14](https://isocpp.org/blog/2015/09/stroustrup-cppcon15-keynote),
- [Herb Sutter, Writting Good C++14… By Default](https://isocpp.org/blog/2015/09/sutter-cppcon15-day2plenary).
