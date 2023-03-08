# First contact with the [Reflection TS](https://en.cppreference.com/w/cpp/experimental/reflect)

I have been interested in [Reflection technology](https://en.wikipedia.org/wiki/Reflective_programming) for some time now and, like many others, I have developed my [own variant](https://github.com/MichaelMiller-/sec21/wiki/Reflection) of it. Now that the Reflection TS is available, I wanted to try it out and share my first experience. Based on the proposal [N4856](https://wg21.link/n4856) the implementation in the expermiental branch was started. An alternative proposal is the document [P1240R2](https://wg21.link/p1240r2). This was also the idea to develop a generic function that converts enum names into strings.

## Current approch

In my current approach, there are two functions that need to be overloaded or specialised for the corresponding enum. This has the advantage that I can use the functions in the generic code without knowing the real implementation. Only at a later stage the [ADL](https://en.cppreference.com/w/cpp/language/adl) solve this problem.

```cpp
// enum_to_string.h

template <typename T>
constexpr auto to_string(T);

template <typename T>
constexpr auto to_enum(std::string_view);
```

This approach also has the advantage that the header is very lightweight.


```cpp
enum class RGB { Red, Green, Blue };

constexpr auto to_string(RGB value)
{
   if (value == RGB::Red) {
      return "Red";
   }
   if (value == RGB::Green) {
      return "Green";
   }
   if (value == RGB::Blue) {
      return "Blue";
   }
   // throw error or return something else
}

template <>
auto to_enum<RGB>(std::string_view value) {
   // ... parse string and convert it to the enum value
}
```

But how about saving all this error-prone code and replacing it with a generated and above all secure variant? This is where the [Reflection TS](https://en.cppreference.com/w/cpp/experimental/reflect) comes into play.

## With [Reflection TS](https://en.cppreference.com/w/cpp/experimental/reflect)

As mentioned above, the proposal [P1240R2](https://wg21.link/p1240r2) gave me the idea to use reflections for the problem. For this, a compile-time known std::array is generated with the function ```enum_lookup()```. The elements of the array are pairs of enum-constant and their corresponding enum-name. The rest of the code only finds for the corresponding enum value. Note that ```to_string``` cannot throw an exception because there is a matching entry for each enum-value in the array.

### Full example
```cpp
#include <functional>
#include <array>
#include <algorithm>
#include <string_view>
#include <iostream>
#include <experimental/reflect>

namespace detail
{
    template <template <typename> typename Transform, std::experimental::reflect::ObjectSequence Sequence, auto... Is>
    consteval auto make_object_sequence_array(std::index_sequence<Is...>) noexcept
    {
        return std::array{Transform<std::experimental::reflect::get_element_t<Is, Sequence>>::value...};
    }

    template <typename T>
        requires std::experimental::reflect::Constant<T> && std::experimental::reflect::Named<T>
    struct constant_and_name
    {
        static constexpr auto value = std::pair{
            std::experimental::reflect::get_constant_v<T>, std::experimental::reflect::get_name_v<T>};
    };

    template <typename T>
    consteval auto enum_lookup() noexcept
        requires std::is_enum_v<T>
    {
        using reflection_t = reflexpr(T);
        using enumerators_t = std::experimental::reflect::get_enumerators_t<reflection_t>;

        return make_object_sequence_array<constant_and_name, enumerators_t>(
            std::make_index_sequence<std::experimental::reflect::get_size_v<enumerators_t>>{});
    }
} // namespace detail

template <typename T>
[[nodiscard]] constexpr auto to_string(T value) noexcept
    requires std::is_enum_v<T>
{
    constexpr auto values = detail::enum_lookup<T>();
    const auto it = std::find_if(begin(values), end(values), [value](auto const& e) { return e.first == value; });
    return it->second;
}

template <typename T, typename Compare = std::equal_to<>>
[[nodiscard]] constexpr auto to_enum(std::string_view value) 
    requires std::is_enum_v<T>
{
    constexpr auto values = detail::enum_lookup<T>();
    const auto it =
        std::find_if(begin(values), end(values), [value](auto const& e) { return Compare{}(e.second, value); });
    if (it == end(values)) {
        throw std::runtime_error("cannot convert string to enum");
    }
    return it->first;
}

// user code
enum class DaysOfTheWeek
{
    Monday,
    Tuesday,
    Wednesday,
    Thursday,
    Friday,
    Saturday,
    Sunday
};

struct case_insensitive_compare
{
    bool operator()(std::string_view lhs, std::string_view rhs) 
    {
      return std::equal(begin(lhs), end(lhs), begin(rhs), end(rhs),
                [](char a, char b) { return tolower(a) == tolower(b); });
    }
};

int main()
{
    // print sime examples
    std::cout << to_string(DaysOfTheWeek::Monday) << std::endl;
    std::cout << to_string(DaysOfTheWeek::Sunday) << std::endl;
    std::cout << to_string(DaysOfTheWeek::Friday) << std::endl;

    // throws because default string comparison is case-sensitiv
    try {
        auto value = to_enum<DaysOfTheWeek>("friday");        
    } catch(std::exception& ex) {
        std::cout << "caught exception: " << ex.what() << std::endl;
    }

    // another conversion with a custom case-insensitive comparator
    {
        auto value = to_enum<DaysOfTheWeek, case_insensitive_compare>("friday");
        if (value == DaysOfTheWeek::Friday) {
            std::cout << "conversion successful\n";
        }
    }

    return 0;
}
```

Link to the full example: [Compiler explorer](https://compiler-explorer.com/z/zc8x4n1nf)

#
Copyright &copy; 2023 by Michael Miller 









