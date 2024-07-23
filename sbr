// <!--
//
// Copyright (c) 2024 Kris Jusiak (kris at jusiak dot net)
//
// Distributed under the Boost Software License, Version 1.0.
// (See accompanying file LICENSE_1_0.txt or copy at
// http://www.boost.org/LICENSE_1_0.txt)
//
#ifdef README
// -->
[![Boost Licence](http://img.shields.io/badge/license-boost-blue.svg)](http://www.boost.org/LICENSE_1_0.txt)
[![Version](https://badge.fury.io/gh/boost-ext%2Fsbr.svg)](https://github.com/boost-ext/sbr/releases)
[![build](https://img.shields.io/badge/build-blue.svg)](https://godbolt.org/z/v3aj66ab5)
[![Try it online](https://img.shields.io/badge/try%20it-online-blue.svg)](https://godbolt.org/z/G473zqE91)

---------------------------------------

## Static/Const branch library

> https://en.wikipedia.org/wiki/Branch_(computer_science)

### Use cases

> When performance matters
  - `and` branches are known at compile-time
  - `or` branches are not changing often at run-time
      - `and/or` branches are expensive to compute/require memory access
      - `and/or` branches are hard to learn by the hardware branch predictor due to their random nature

> Examples: logging, tracing, configuration, algo, ...

### Features

- Single header (https://raw.githubusercontent.com/boost-ext/sbr/main/sbr - for integration see [FAQ](#faq))
- Minimal [API](#api)
- Verifies itself upon include (can be disabled with `-DNTEST` - see [FAQ](#faq))

### Requirements

- C++20 ([clang++15+, g++10+](https://en.cppreference.com/w/cpp/compiler_support)) / [x86-64](https://en.wikipedia.org/wiki/X86-64) / [Linux](https://en.wikipedia.org/wiki/Linux)

---

### Overview

> `static_branch` - Minimal overhead run-time branch (https://godbolt.org/z/G473zqE91)

```cpp
/**
 * Note: `fun` can be inline/noinline/constexpr/etc.
 * constexpr void fun();
 * inline void fun();
 * [[gnu::noinline]] void fun()
 * [[gnu::always_inline]] void fun()
 */
void fun() {
  if (sbr::static_branch<"semi run-time branch">::get()) {
    std::puts("taken");
  } else {
    std::puts("not taken");
  }
}

int main() {
  fun(); // not taken

  sbr::static_branch<"semi run-time branch">::set(true);
  fun(); // taken

  sbr::static_branch<"semi run-time branch">::set(false);
  fun(); // not taken
}
```

```cpp
main: // $CXX -O3
  lea rdi, [rip + .L.str.1]
  nop # code patching (nop->nop)
  lea rdi, [rip + .L.str.2]
 .Ltmp1:
  call puts@PLT # not taken

  call static_branch<"semi run-time branch">::set(true) # relatively slow

  lea rdi, [rip + .L.str.1]
  jmp .Ltmp2 # code patching (nop->jmp)
  lea rdi, [rip + .L.str.2]
 .Ltmp2:
  call puts@PLT # taken

  call static_branch<"semi run-time branch">::set(false) # relatively slow

  lea rdi, [rip + .L.str.1]
  nop # code patching (jmp->nop)
  lea rdi, [rip + .L.str.2]
 .Ltmp3:
  call puts@PLT # not taken

  xor  eax, eax # return 0
  ret

.L.str.1: .asciz "taken"
.L.str.2: .asciz "not taken"
```

---

> `const_branch` - Zero overhead compile-time branch (https://godbolt.org/z/Kb9oqnef6)

```cpp
template<auto tag = []{}> // Note: `tag` is required to delay `fun` instantiation
auto fun() {
  if constexpr (sbr::const_branch<"compile-time branch">::get<tag>()) {
    std::puts("taken");
  } else {
    std::puts("not taken");
  }
}

int main() {
  fun(); // not taken

  static_assert(sbr::const_branch<"compile-time branch">::set<true>());
  fun(); // taken

  static_assert(sbr::const_branch<"compile-time branch">::set<false>());
  fun(); // not taken
}
```

```cpp
main: // $CXX -O3
  lea  rbx, [rip + .L.str.1]
  mov  rdi, rbx
  call puts@PLT # not taken

  lea  rdi, [rip + .L.str.2]
  call puts@PLT # taken

  mov  rdi, rbx
  call puts@PLT # not taken

  xor  eax, eax # return 0
  ret

.L.str.1: .asciz "not taken"
.L.str.2: .asciz "taken"
```

---

### API

```cpp
/**
 * Minimal overhead named run-time branch (default: not set)
 * @tparam name branch name, ex. static_branch<"branch">
 */
template<fixed_string Name>
struct static_branch {
  static constexpr u64 id = Name.hash();

  /**
   * Updates branch direction (not thread-safe)
   * @tparam protect_policy policy (default: protect)
   * @param direction new branch direction
   * @return true on success, false otherwise
   */
  template<auto protect_policy = protect>
  static constexpr auto set(const bool direction) noexcept -> bool;

  /**
   * Returns current branch direction (thread-safe)
   * @return current branch direction
   */
  [[gnu::always_inline]] [[nodiscard]] static constexpr auto get() noexcept -> bool;
};
```

```cpp
/**
 * Zero overhead named compile-time branch (default: not set)
 * Note: Limited to a single translation unit
 * @tparam name branch name, ex. const_branch<"branch">
 */
template<fixed_string Name>
struct const_branch {
  /**
   * Updates branch direction
   * @param direction new branch direction
   * @return true (always)
   */
  template<bool Direction>
  static consteval auto set() noexcept -> bool;

  /**
   * Returns current branch direction
   * @return current branch direction
   */
  [[nodiscard]] static consteval auto get() noexcept -> bool;
};
```

> Configuration

```cpp
#define SBR 1'0'1 // Current library version (SemVer)
```

---

### FAQ

- How does it work?

  > `sbr` is using technique called code patching - which basically means that the code modifies itself.

  `static_branch` is based on https://docs.kernel.org/staging/static-keys.html and it requires `asm goto` support (gcc, clang).
  `sbr` currently supports x86-64 Linux, but other platforms can be added using the same technique.

  Example:

    ```cpp
    if (sbr::static_branch<"how does it work?">::get()) {
      return 42;
    } else {
      return 0;
    }
    ```

  it will emit...

    ```cpp
    main:
      .byte 15 31 68 0 0 # nop - https://www.felixcloutier.com/x86/nop
      xor eax, eax # return 0
      ret
    .LBB1:
      mov eax, 42 # return 42
      ret
    ```

  which will effectively execute (static branch is set to false by default)...

    ```cpp
    main:
      nop
      xor eax, eax # return 0
      ret
    ```

  now, if we change the branch direction (at run-time / before or after the branch)...

    ```cpp
    sbr::static_branch<"how does it work?">::set(true);

    if (sbr::static_branch<"how does it work?">::get()) {
      return 42;
    } else {
      return 0;
    }
    ```

  it will emit...

    ```cpp
    main:
      # relatively slow, it will 'code patch' the new branch direction
      call sbr::static_branch<"how does it work?">::set(true); # nop->jmp or jmp->nop

      jmp .LBB1: (nop->jmp - changed in the memory of the program via `set(true)` call)
      xor eax, eax # return 0
      ret
    .LBB1:
      mov eax, 42 # return 42
      ret
    ```

  `const_branch` is using stateful template meta-programming (via friend injection) to maintain different branch versions.

- How to integrate with CMake/CPM?

    ```
    CPMAddPackage(
      Name sb
      GITHUB_REPOSITORY boost-ext/sb
      GIT_TAG v1.0.1
    )
    add_library(mp INTERFACE)
    target_include_directories(mp SYSTEM INTERFACE ${mp_SOURCE_DIR})
    add_library(sbr::sb ALIAS sb)
    ```

    ```
    target_link_libraries(${PROJECT_NAME} sbr::sb);
    ```

- Acknowledgments

  > https://docs.kernel.org/staging/static-keys.html, https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html, https://www.agner.org/optimize/instruction_tables.pdf, https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html, https://www.felixcloutier.com/documents/gcc-asm.html, https://www.felixcloutier.com/x86, https://uops.info/table.html, https://arxiv.org/abs/2308.14185, https://arxiv.org/pdf/2011.13127

<!--
#else
#ifndef SBR
#define SBR 1'0'1 // SemVer
#pragma GCC system_header

#include <cstring>    // std::memcpy
#include <unistd.h>   // PAGESIZE
#include <sys/mman.h> // mprotect

namespace sbr::inline v1_0_1 {
using u8  = __UINT8_TYPE__;
using u16 = __UINT16_TYPE__;
using u32 = __UINT32_TYPE__;
using u64 = __UINT64_TYPE__;
struct entry { u64 id{}; u64 code{}; u64 offset{}; u64 state{}; };
template<class T, u32 N>
struct fixed_string {
  consteval explicit(false) fixed_string(const T (&str)[N]) noexcept {
    for (u32 i = 0u; i < N; ++i) { data[i] = str[i]; }
  }
  [[nodiscard]] consteval auto size() const noexcept { return N; }
  [[nodiscard]] consteval auto hash() const noexcept -> u32 {
    constexpr u32 FNV_OFFSET = 216613626u;
    constexpr u32 FNV_32_PRIME = 16777619u;
    u32 hash = FNV_OFFSET;
    for (auto i = 0u; i < N; ++i) {
      hash ^= static_cast<u32>(data[i]);
      hash *= FNV_32_PRIME;
    }
    return hash;
  }
  T data[N]{};
};
template<class T, u32 N> fixed_string(const T (&str)[N]) -> fixed_string<T, N>;
} // sb
extern sbr::entry __start___sb [[gnu::section("__sb")]];
extern sbr::entry __stop___sb [[gnu::section("__sb")]];
namespace sbr::inline v1_0_1 {
inline namespace x86 {
  namespace detail {
  static constexpr u8 JMP[]{0xe9}; // https://www.felixcloutier.com/x86/jmp
  static constexpr u8 NOP[]{0x0f, 0x1f, 0x44, 0x00, 0x00}; // https://www.felixcloutier.com/x86/nop
  static constexpr void (*copy[])(const entry*){
    [](const auto* entry) { std::memcpy(reinterpret_cast<void*>(entry->code), &NOP, sizeof(NOP)); },
    [](const auto* entry) {
      struct [[gnu::packed]] { u8 op; u32 offset; } jmp{.op = JMP[0], .offset = static_cast<u32>(entry->offset)};
      static_assert(sizeof(jmp) == sizeof(NOP));
      std::memcpy(reinterpret_cast<void*>(entry->code), &jmp, sizeof(jmp));
    }
  };
  } // detail

  static constexpr auto protect = [](entry* entry, u32 size, u32 permissions = PROT_READ | PROT_WRITE | PROT_EXEC) {
    if (static thread_local const auto page_size = sysconf(_SC_PAGESIZE); entry->state != permissions and
      mprotect(reinterpret_cast<void*>(entry->code & ~(page_size - 1u)), size, permissions)) {
      return false;
    }
    entry->state = permissions;
    return true;
  };

  /**
   * Minimal overhead named run-time branch (default: not set)
   * @tparam name branch name, ex. static_branch<"branch">
   */
  template<fixed_string Name>
  struct static_branch {
    static constexpr u64 id = Name.hash();

    /**
     * Updates branch direction (not thread-safe)
     * @tparam protect_policy policy (default: protect)
     * @param direction new branch direction
     * @return true on success, false otherwise
     */
    template<auto protect_policy = protect>
    static constexpr auto set(const bool direction) noexcept -> bool {
      for (entry* entry = &__start___sb; entry < &__stop___sb; ++entry) {
        if (entry->id != id) continue;
        if (not protect_policy(entry, sizeof(detail::NOP))) return false;
        detail::copy[direction](entry);
      }
      return true;
    }

    /**
     * Returns current branch direction (thread-safe)
     * @return current branch direction
     */
    [[gnu::always_inline]] [[nodiscard]] static constexpr auto get() noexcept -> bool {
      using detail::NOP;
      asm volatile goto("0:"
        ".byte %c3,%c4,%c5,%c6,%c7 \n"
        ".pushsection __sb, \"aw\" \n"
        ".balign %c0 \n"
        ".quad %c1, 0b, %l[true_] - (0b + %c2), 0 \n"
        ".popsection \n"
        : : "i"(alignof(u64)),
            "i"(id),
            "i"(sizeof(NOP)),
            "i"(NOP[0]), "i"(NOP[1]), "i"(NOP[2]), "i"(NOP[3]), "i"(NOP[4])
        : : true_);
      false_: return false;
      true_:  return true;
    }
  };

  /**
   * Zero overhead named compile-time branch (default: not set)
   * Note: Limited to a single translation unit
   * @tparam name branch name, ex. const_branch<"branch">
   */
  template<fixed_string Name>
  struct const_branch {
    template<u32> struct version { constexpr auto friend $get(version); };
    template<u32 V, bool R> struct new_version { constexpr auto friend $get(version<V>) { return R; } };

    /**
     * Updates branch direction
     * @param direction new branch direction
     * @return true
     */
    template<bool Direction, auto tag = []{}, u32 N = 0u>
    static consteval auto set() noexcept {
      if constexpr (requires { $get(version<N>{}); }) {
        return set<Direction, tag, N + 1u>();
      } else {
        return new_version<N, Direction>{}, true;
      }
    }

    /**
     * Returns current branch direction
     * @return current branch direction
     */
    template<auto tag, u32 N = 0u>
    [[nodiscard]] static consteval auto get() noexcept -> bool {
      if constexpr (requires { $get(version<N>{}); }) {
        return get<tag, N + 1u>();
      } else if constexpr (N > 0u) {
        return $get(version<N - 1u>{});
      } else {
        return false;
      }
    }
  };
} // namespace x86
} // namespace sb

#ifndef NTEST
static_assert(([] {
  // sbr::fixed_string
  {
    static_assert(sizeof("") == sbr::fixed_string{""}.size());
    static_assert(sizeof("x86") == sbr::fixed_string{"x86"}.size());
    static_assert(sizeof("arm64") != sbr::fixed_string{"x86"}.size());
    static_assert(sbr::fixed_string{"x86"}.hash() == sbr::fixed_string{"x86"}.hash());
    static_assert(sbr::fixed_string{"arm64"}.hash() != sbr::fixed_string{"x86"}.hash());
  }

  // sbr::static_branch
  {
    static_assert(sbr::static_branch<"::sbr::x86">::id == sbr::static_branch<"::sbr::x86">::id);
    static_assert(sbr::static_branch<"::sbr::arm64">::id != sbr::static_branch<"::sbr::x86">::id);
  }

  // sbr::const_branch
  {
    static_assert(not sbr::const_branch<"::sbr::compile-time">::get<[]{}>());
    static_assert(sbr::const_branch<"::sbr::compile-time">::set<true, []{}>());
    static_assert(sbr::const_branch<"::sbr::compile-time">::get<[]{}>());
    static_assert(sbr::const_branch<"::sbr::compile-time">::set<false, []{}>());
    static_assert(not sbr::const_branch<"::sbr::compile-time">::get<[]{}>());
  }
}(), true));
#endif // NTEST
#endif // SBR
#endif // README
