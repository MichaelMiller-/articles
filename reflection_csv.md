# CSV writer with [Reflection TS](https://en.cppreference.com/w/cpp/experimental/reflect)

In my [previous article](https://github.com/MichaelMiller-/articles/blob/main/enum_to_string.md), I showed how the [Reflection TS](https://en.cppreference.com/w/cpp/experimental/reflect) can be used to convert an enumeration into string literals. But what about whole records? Can the Reflection TS be easily used here as well? Let's make a simple example. Let's assume I have a sequenced container that holds any data structure. I want to output this as a [CSV](https://en.wikipedia.org/wiki/Comma-separated_values) file.

To do this, we write a small class that encapsulates the actual functionality. The class gets an iterator pair in its constructor that describes the range of the data.
```cpp
class csv_file
{
   std::string file_content{};
public:
   template <typename Iterator>
   explicit csv_file(Iterator first, Iterator last)
      : file_content{generate_content<typename std::iterator_traits<Iterator>::value_type>(first, last)}
   {}

   [[nodiscard]] auto content() const noexcept { return file_content; }
};
```

The actual worker is the function ``generate_content()```. This is called in addition to the iterators with an explicit template-parameter that corresponds to the type that the iterators manage. In the function body, a reflection of type T is retrieved and then processed. First, a tuple of reflection objects is created from all public data members.

```cpp
template <typename T, typename Iterator>
auto generate_content(Iterator first, Iterator last) noexcept -> std::string
{
   namespace reflect = std::experimental::reflect;

   using reflection_t = reflexpr(T);
   using members_t = reflect::get_public_data_members_t<reflection_t>;

   constexpr auto members = make_object_sequence_tuple<members_t>(std::make_index_sequence<reflect::get_size_v<members_t>>{});
   // ...
}

```cpp
template <std::experimental::reflect::ObjectSequence Sequence, auto... Is>
consteval auto make_object_sequence_tuple(std::index_sequence<Is...>) noexcept
{
   return std::tuple<std::experimental::reflect::get_element_t<Is, Sequence>...>{};
}
```

From this tuple, information can be extracted very easily, such as the name of the member or a pointer to it.
```cpp
constexpr auto member_names = std::apply([]<typename... Ts>(Ts...) { return std::make_tuple(reflect::get_name_v<Ts>...); }, members);
constexpr auto member_pointers = std::apply([]<typename... Ts>(Ts...) { return std::make_tuple(reflect::get_pointer_v<Ts>...); }, members);
```

Afterwards, this information only has to be written into a stream. The ``member_names`` are the column descriptions and the pointers are needed to access the data.

```cpp
std::stringstream stream{};
// transforms a std::tuple of names with the given seperator into a std::string
stream << concat(member_names, ", ") << std::endl;

for (; first != last; ++first) {
   const auto values = std::apply([value = *first](auto const&... member) { return std::make_tuple(value.*member...); }, member_pointers);
   // transforms a std::tuple of values with the given seperator into a std::string
   stream << concat(values, ", ") << std::endl;
}
```

A few more small functions to simplify the handling of the above-mentioned class.

```cpp
std::ostream& operator<<(std::ostream& os, csv_file const& obj) { return os << obj.content(); }

template <typename Container>
inline auto csv(Container const& container)
{
   return csv_file{begin(container), end(container)};
}
```

Finally, the whole code looks like this.

## Example
```cpp
struct user
{
   std::string name{};
   std::string password{};
   int karma{};
   double cash{};
};

int main()
{
   auto users = std::vector<user>{
      { "John Doe", "secret", 42, 0 },
      { "Max Mustermann", "****", 1, 45'678 }
   };

   std::cout << csv(users) << std::endl;
}
```

The output of the code above is:
```
name, password, karma, cash
John Doe, secret, 42, 0
Max Mustermann, ****, 1, 45678
```

## Summary

This experiment was also very easy to implement with the [Reflection TS](https://en.cppreference.com/w/cpp/experimental/reflect). More will follow in any case.

Finally, I would like to refer to my new library: [out](https://github.com/MichaelMiller-/out). It is based on the investigations and experiences I have made with  [Reflection TS](https://en.cppreference.com/w/cpp/experimental/reflect). You can also find all the code for this article there. Feel free to give me feedback.

#
Copyright &copy; 2023 by Michael Miller 
