# MsgPack

MessagePack implementation for Arduino (compatible with other C++ apps)


## Typical Usage

This library is only for serialize / deserialize.
To send / receive serialized data with `Stream` class, please use [MsgPacketizer](https://github.com/hideakitai/MsgPacketizer).

``` C++
#include <MsgPack.h>

// input to msgpack
int i = 123;
float f = 1.23;
MsgPack::str_t s = "str"; // std::string or String
MsgPack::arr_t<int> v {1, 2, 3}; // std::vector or arx::vector
MsgPack::map_t<String, float> m {{"one", 1.1}, {"two", 2.2}, {"three", 3.3}}; // std::map or arx::map

// output from msgpack
int ri;
float rf;
MsgPack::str_t rs;
MsgPack::arr_t<int> rv;
MsgPack::map_t<String, float> rm;

void setup()
{
    delay(2000);
    Serial.begin(115200);
    Serial.println("msgpack test start");

    // serialize to msgpack
    MsgPack::Packer packer;
    packer.serialize(i, f, s, v, m);

    // deserialize from msgpack
    MsgPack::Unpacker unpacker;
    unpacker.feed(packer.data(), packer.size());
    unpacker.deserialize(ri, rf, rs, rv, rm);

    if (i != ri) Serial.println("failed: int");
    if (f != rf) Serial.println("failed: float");
    if (s != rs) Serial.println("failed: string");
    if (v != rv) Serial.println("failed: vector<int>");
    if (m != rm) Serial.println("failed: map<string, int>");

    Serial.println("msgpack test success");
}

void loop()
{
}
```

## Custom Class Adaptation

To serialize / deserialize custom type you defined, please use `MSGPACK_DEFINE()` macro inside of your class.

``` C++
struct CustomClass
{
    int i;
    float f;
    MsgPack::str_t s;

    MSGPACK_DEFINE(i, f, s);
};
```

After that, you can pack your class completely same as other types.

``` C++
int i;
float f;
MsgPack::str_t s;
CustomClass c;

MsgPack::Packer packer;
packer.serialize(i, f, s, c); // -> packer.serialize(i, f, s, c.i, c.f, c.s)
```

### Custom Class with Inheritance

Also you can use `MSGPACK_BASE()` macro to pack values of base class.

``` C++
struct Base
{
    int i;
    float f;

    MSGPACK_DEFINE(i, f);
};

struct Derived : public Base
{
    MsgPack::str_t s;

    MSGPACK_DEFINE(s, MSGPACK_BASE(Base)); // -> packer.serialize(s, Base::i, Base::f)
};
```


## Supported Type Adaptors

These are the lists of types which can be `serialize` and `deserialize`.
You can also `pack()` variable one by one.

### NIL

- `MsgPack::object::nil_t`

### Bool

- `bool`

### Integer

- `char (signed/unsigned)`
- `ints (signed/unsigned)`

### Float

- `float`
- `double`

### Str

- `char*`
- `char[]`
- `std::string` or `String(Arduino)` (`MsgPack::str_t`)

### Bin

- `unsigned char*` (need to `serialize(ptr, size)` or `pack(ptr, size)`)
- `unsigned char[]` (need to `serialize(ptr, size)` or `pack(ptr, size)`)
- `std::vector<char>` (`MsgPack::bin_t<char>`)
- `std::vector<unsigned char>` (`MsgPack::bin_t<unsigned char>`)
- `std::array<char>`
- `std::array<unsigned char>`

### Array

- `T[]` (need to `serialize(ptr, size)` or `pack(ptr, size)`)
- `std::vector` (`MsgPack::arr_t<T>`)
- `std::array`
- `std::deque`
- `std::pair`
- `std::tuple`
- `std::list`
- `std::forward_list`
- `std::set`
- `std::multiset`
- `std::unordered_set`
- `std::unordered_multiset`

### Map

- `std::map` (`MsgPack::map_t<T>`)
- `std::multimap`
- `std::unordered_map`
- `std::unordered_multimap`

### Ext

- `MsgPack::object::ext`

### TimeStamp

- `MsgPack::object::timespec`

### N/A

- `std::queue`
- `std::priority_queue`
- `std::bitset`
- `std::stack`


### Note

- `unordered_xxx` cannot be used in all Arduino
- C-style array and pointers are supported only packing.
- for NO-STL Arduino, following types can be used
  - all types of NIL, Bool, Integer, Float, Str, Bin
  - for Array, only `T[]` and `MsgPack::arr_t<T>` (= `arx::vector<T>`) can be used
  - for Map, only `MsgPack::map_t<T, U>` (= `arx::map<T, U>`) can be used
  - for the detail of `arx::xxx`, see [ArxContainer](https://github.com/hideakitai/ArxContainer)


### Additional Types for MsgPack

There are some additional types are defined for compatibility to no-stl Arduino and other general C++ apps.

#### Type Aliases for Str / Bin / Array / Map

For no-stl Arduino, these type aliases are defined.
Please see "Memory Management" section and [ArxContainer](https://github.com/hideakitai/ArxContainer) for detail.

- `MsgPack::str_t` = `String`
- `MsgPack::bin_t<T>` = `arx::vector<T, N = MSGPACK_MAX_PACKET_BYTE_SIZE>`
- `MsgPack::arr_t<T>` = `arx::vector<T, N = MSGPACK_MAX_ARRAY_SIZE>`
- `MsgPack::map_t<T, U>` = `arx::map<T, U, N = MSGPACK_MAX_MAP_SIZE>`

For other stl-enabled Arduino, these types are:

- `MsgPack::str_t` = `String`
- `MsgPack::bin_t<T>` = `std::vector<T>`
- `MsgPack::arr_t<T>` = `std::vector<T>`
- `MsgPack::map_t<T, U>` = `std::map<T, U>`

For general C++ apps, only difference is:

- `MsgPack::str_t` = `std::string`


#### MsgPack::obeject::nil_t

`MsgPack::object::nil_t` is used to `pack` and `unpack` Nil type.
This object is just a dummy and do nothing.

#### MsgPack::obeject::ext

`MsgPack::object::ext` holds binary data of Ext type.

``` C++
// create ext type with args: int8_t, const uint8_t*, uint32_t
MsgPack::object::ext e(type, bin_ptr, size);
MsgPack::Packer packer;
packer.serialize(e); // serialize ext type

MsgPack::object::ext r;
msgPack::Unpacker unpacker;
unpacker.feed(packer.data(), packer.size());
unpacker.deserialize(r); // deserialize ext type
```


#### MsgPack::obeject::timespec

`MsgPack::object::timespec` is used to `pack` and `unpack` Timestamp type.

``` C++
MsgPack::object::timespec t = {
    .tv_sec  = 123456789, /* int64_t  */
    .tv_usec = 123456789  /* uint32_t */
};
MsgPack::Packer packer;
packer.serialize(t); // serialize timestamp type

MsgPack::object::timespec r;
msgPack::Unpacker unpacker;
unpacker.feed(packer.data(), packer.size());
unpacker.deserialize(r); // deserialize timestamp type
```


## Other Options

### Packet Data Storage Class Inside

STL is used to handle packet data by default, but for following boards/architectures, [ArxContainer](https://github.com/hideakitai/ArxContainer) is used to store the packet data because STL can not be used for such boards.
The storage size of such boards for max packet binary size and number of msgpack objects are limited.

- AVR
- megaAVR
- SAMD
- SPRESENSE


### Memory Management (for NO-STL Boards)

As mentioned above, for such boards like Arduino Uno, the storage sizes are limited.
And of course you can manage them by defining following macros.
But these default values are optimized for such boards, please be careful not to excess your boards storage/memory.

``` C++
// msgpack serialized binary size
#define MSGPACK_MAX_PACKET_BYTE_SIZE  128
// max size of MsgPack::arr_t
#define MSGPACK_MAX_ARRAY_SIZE          8
// max size of MsgPack::map_t
#define MSGPACK_MAX_MAP_SIZE            8
// msgpack objects size in one packet
#define MSGPACK_MAX_OBJECT_SIZE        24
```

These macros have no effect for STL enabled boards.


### STL library for Arduino Support

For such boards, there are several STL libraries, like [ArduinoSTL](https://github.com/mike-matera/ArduinoSTL), [StandardCPlusPlus](https://github.com/maniacbug/StandardCplusplus), and so on.
But such libraries are mainly based on [uClibc++](https://cxx.uclibc.org/) and it has many lack of function.
I considered to support them but I won't support them unless uClibc++ becomes much better compatibility to standard C++ library.
I reccomend to use low cost but much better performance chip like ESP series.


## Embedded Libraries

- [ArxTypeTraits v0.1.6](https://github.com/hideakitai/ArxTypeTraits)
- [ArxContainer v0.3.4](https://github.com/hideakitai/ArxContainer)
- [DebugLog v0.1.4](https://github.com/hideakitai/DebugLog)
- [TeensyDirtySTLErrorSolution v0.1.0](https://github.com/hideakitai/TeensyDirtySTLErrorSolution)


## Used Inside of

- [MsgPacketizer](https://github.com/hideakitai/MsgPacketizer)


## APIs

### MsgPack::Packer

``` C++
template <typename First, typename ...Rest>
void serialize(const First& first, Rest&&... rest)
template <typename T>
void serialize(const T* data, const size_t size) // only for poitner types

void pack<T>(const T& t)
void pack<T>(const T* ptr, const size_t size) // only for pointer types

const bin_t<uint8_t>& packet() const
const uint8_t* data() const
size_t size() const
void clear()

void packNil()
void packNil(const object::nil_t& n)
void packBool(const bool b)
void packUInt7(const uint8_t value)
void packUInt8(const uint8_t value)
void packUInt16(const uint16_t value)
void packUInt32(const uint32_t value)
void packUInt64(const uint64_t value)
void packInt5(const int8_t value)
void packInt8(const int8_t value)
void packInt16(const int16_t value)
void packInt32(const int32_t value)
void packInt64(const int64_t value)
void packFloat32(const float value)
void packFloat64(const double value)
void packString5(const str_t& str)
void packString5(const char* value)
void packString8(const str_t& str)
void packString8(const char* value)
void packString16(const str_t& str)
void packString16(const char* value)
void packString32(const str_t& str)
void packString32(const char* value)
void packBinary8(const uint8_t* value, const uint8_t size)
void packBinary16(const uint8_t* value, const uint16_t size)
void packBinary32(const uint8_t* value, const uint32_t size)
void packArraySize(const size_t size)
void packArraySize4(const uint8_t value)
void packArraySize16(const uint16_t value)
void packArraySize32(const uint32_t value)
void packMapSize(const size_t size)
void packMapSize4(const uint8_t value)
void packMapSize16(const uint16_t value)
void packMapSize32(const uint32_t value)
void packFixExt1(const int8_t type, const uint8_t value)
void packFixExt2(const int8_t type, const uint16_t value)
void packFixExt2(const int8_t type, const uint8_t* ptr)
void packFixExt2(const int8_t type, const uint16_t* ptr)
void packFixExt4(const int8_t type, const uint32_t value)
void packFixExt4(const int8_t type, const uint8_t* ptr)
void packFixExt4(const int8_t type, const uint32_t* ptr)
void packFixExt8(const int8_t type, const uint64_t value)
void packFixExt8(const int8_t type, const uint8_t* ptr)
void packFixExt8(const int8_t type, const uint64_t* ptr)
void packFixExt16(const int8_t type, const uint64_t value_h, const uint64_t value_l)
void packFixExt16(const int8_t type, const uint8_t* ptr)
void packFixExt16(const int8_t type, const uint64_t* ptr)
template <typename T>
void packFixExt(const int8_t type, const T value)
void packFixExt(const int8_t type, const uint64_t value_h, const uint64_t value_l)
void packFixExt(const int8_t type, const uint8_t* ptr, const uint8_t size)
void packFixExt(const int8_t type, const uint16_t* ptr, const uint8_t size)
void packFixExt(const int8_t type, const uint32_t* ptr, const uint8_t size)
void packFixExt(const int8_t type, const uint64_t* ptr, const uint8_t size)
void packExtSize8(const int8_t type, const uint8_t size)
void packExtSize16(const int8_t type, const uint16_t size)
void packExtSize32(const int8_t type, const uint32_t size)
template <typename T, typename U>
void packExt(const int8_t type, const T* ptr, const U size)
void packExt(const object::ext& e)
void packTimestamp32(const uint32_t unix_time_sec)
void packTimestamp64(const uint64_t unix_time)
void packTimestamp64(const uint64_t unix_time_sec, const uint32_t unix_time_nsec)
void packTimestamp96(const int64_t unix_time_sec, const uint32_t unix_time_nsec)
void packTimestamp(const object::timespec& time)
```

### MsgPack::Unpacker

``` C++
bool feed(const uint8_t* data, size_t size)

template <typename First, typename ...Rest>
void deserialize(First& first, Rest&&... rest)
template <typename... Ts>
void deserializeToTuple(std::tuple<Ts...>& t)

template <typename T>
void unpack(T& value)

template <typename T>
bool unpackable(const T& value) const

bool available() const
size_t size() const
void index(const size_t i)
size_t index() const
void clear()

bool unpackNil()
bool unpackBool()
uint8_t unpackUInt7()
uint8_t unpackUInt8()
uint16_t unpackUInt16()
uint32_t unpackUInt32()
uint64_t unpackUInt64()
int8_t unpackInt5()
int8_t unpackInt8()
int16_t unpackInt16()
int32_t unpackInt32()
int64_t unpackInt64()
float unpackFloat32()
double unpackFloat64()
str_t unpackString5()
str_t unpackString8()
str_t unpackString16()
str_t unpackString32()
template <typename T = uint8_t>
bin_t<T> unpackBinary()
template <typename T = uint8_t>
bin_t<T> unpackBinary8()
template <typename T = uint8_t>
bin_t<T> unpackBinary16()
template <typename T = uint8_t>
bin_t<T> unpackBinary32()
template <typename T, size_t N>
std::array<T, N> unpackBinary()
template <typename T, size_t N>
std::array<T, N> unpackBinary8()
template <typename T, size_t N>
std::array<T, N> unpackBinary16()
template <typename T, size_t N>
std::array<T, N> unpackBinary32()
size_t unpackArraySize()
size_t unpackMapSize()
object::ext unpackExt()
object::timespec unpackTimestamp()

bool isNil() const
bool isBool() const
bool isUInt7() const
bool isUInt8() const
bool isUInt16() const
bool isUInt32() const
bool isUInt64() const
bool isUInt() const
bool isInt5() const
bool isInt8() const
bool isInt16() const
bool isInt32() const
bool isInt64() const
bool isInt() const
bool isFloat32() const
bool isFloat64() const
bool isFloat() const
bool isStr5() const
bool isStr8() const
bool isStr16() const
bool isStr32() const
bool isStr() const
bool isBin8() const
bool isBin16() const
bool isBin32() const
bool isBin() const
bool isArray4() const
bool isArray16() const
bool isArray32() const
bool isArray() const
bool isMap4() const
bool isMap16() const
bool isMap32() const
bool isMap() const
bool isFixExt1() const
bool isFixExt2() const
bool isFixExt4() const
bool isFixExt8() const
bool isFixExt16() const
bool isFixExt() const
bool isExt8() const
bool isExt16() const
bool isExt32() const
bool isExt() const
bool isTimestamp32() const
bool isTimestamp64() const
bool isTimestamp96() const
bool isTimestamp() const
```

## Reference

- [MessagePack Specification](https://github.com/msgpack/msgpack/blob/master/spec.md)
- [msgpack adaptor](https://github.com/msgpack/msgpack-c/wiki/v2_0_cpp_adaptor)
- [msgpack object](https://github.com/msgpack/msgpack-c/wiki/v2_0_cpp_object)
- [msgpack-c wiki](https://github.com/msgpack/msgpack-c/wiki)


## License

MIT
