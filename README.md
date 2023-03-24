# Introduction

This documentation covers how to deserialize a replay file into readible data. It does *not* cover how to interpret that data. The below information is valid as of Rocket League version `2.23`.

A replay file is split into three main sections: the header, which includes some data about the match overall, the body, which is essentially just the complete network stream sent from the Psyonix servers to the client, and the footer, which contains data that is key to being able to successfully deserialize the body.

This documentation is split up into multiple sections:

1. [Reading Bits and Bytes](#reading-bits-and-bytes) - Basics of a little endian bit stream.
2. [Types and Structures](#types-and-structures) - Details on how to deserialize the data types and data structures used throughout the replay.
    1. [Core Types](#core-types) - Core data types.
    2. [Basic Data Structures](#basic-data-structures) - Basic (non-attribute) data structures.
    3. [Advanced Data Structures (Attributes)](#advanced-data-structures-attributes) - Attribute data structures.
3. [Replay Deserialization - Basic Data](#replay-deserialization---step-1) - Basic structure of a replay file, and how to deserialize the header, beginning of the body, and the footer.
    1. [Deserializing the Header](#deserializing-the-header) - How to deserialize the header.
    2. [Deserializing the Body](#deserializing-the-body) - How to deserialize the beginning of the body.
    3. [Deserializing the Footer](#deserializing-the-footer) - How to deserialize the footer.
4. [Replay Deserialization - Network Stream Preparation](#replay-deserialization---network-stream-preparation) - Making preparations for the deserialization of the network stream.
    1. [Preparation of the Class Net Cache](#preparation-of-the-class-net-cache) - Updating the class net cache and modifying it to our needs.
    2. [Preparation of Miscellaneous Information](#preparation-of-miscellaneous-information) - Preparing various pieces of information that will be used later on.
5. [Replay Deserialization - Network Stream](#replay-deserialization---network-stream) - Deserializing the network stream.
    1. [Deserializing a New Actor](#deserializing-a-new-actor) - How to deserialize a new actor.
    2. [Deserializing a Deleted Actor](#deserializing-a-deleted-actor) - How to deserialize a deleted actor.
    3. [Deserializing an Updated Actor](#deserializing-an-updated-actor) - How to deserialize an updated actor.
5. [Hash Maps](#hash-maps) - Various hash maps used through the network stream deserialization process.
    1. [Object:SpawnTrajectory](#objectspawntrajectory) - A hash map of object names to their corresponding spawn trajectories.
    2. [Object:AttributeType](#objectattributetype) - A hash map of object names to their corresponding attribute types.
    3. [Class:ParentClass](#classparentclass) - A hash map of class names to their corresponding parent classes.
    4. [Object:Parent](#objectparent) - A hash map of object names to their corresponding parent object names.
6. [Unknowns](#unknowns) - A list of all currently unknown portions of the replay format.
7. [Acknowledgements](#acknowledgements) - Final acknowledgements.

# Reading Bits and Bytes

All numbers should be read in little endian. That's simple enough, but when it comes to reading bits in little endian (and yes, unfortunately you will be reading about 95% of the replay bit-by-bit), it becomes a lot more complicated. Below I will explain the bit order and how to properly read bits from the stream:

Suppose we have an 8-bit sequence (a byte) stored in our bit buffer:
```
01101001
```
and we need 5 bits from that sequence. We will consume 5 bits from the end (instead of from the beginning, like one might normally do), like so:
```
01101001
   ^^^^^
```
which gives us `01001`, or `9`. Suppose the next byte in the stream is `11000101`, we add it to the beginning of the buffer instead of the end (i.e. left shift the new byte by the remaining number of bits in the buffer, then merge with the buffer):
```
11000101 011
         ^
          remainder from the previous data in the buffer
```
Reading a byte from the buffer simply requires reading 8 bits. Reading multiple bytes should be done in sequence (e.g. you shouldn't read 16 bits from the buffer, then split into two bytes), although it can be done so long as you reverse the byte order afterward. Decoding numbers from the bit buffer is as simple as reading `N` byte segments from the buffer, then decoding the number in little endian as per usual. When reading unsigned integers, the speed can be slightly improved by simply reading `N` bits from the buffer (e.g. `32` bits for a `32-bit unsigned integer`), rather than reading bits as bytes, then converting those bytes to an unsigned integer.

# Types and Structures

Below is some shorthand for certain data types and data structures that are used numerous times throughout the replay deserialization process.

Something you will see a few times are mentions of `ENGINE_VERSION`, `LICENSEE_VERSION`, `NET_VERSION`, and `IS_RL_223`. These are values in the header (or derived from the header, in the case of `IS_RL_223`) that in some cases determine the method of deserialization for certain types/structures. How `IS_RL_223` is calculated will be covered in [Preparation of Miscellaneous Information](#preparation-of-miscellaneous-information).

## Core Types

Core data types.

- `u8` > An unsigned 8-bit integer.
- `u32` > An unsigned 32-bit integer.
- `u64` > An unsigned 64-bit integer.
- `i32` > A signed 32-bit integer.
- `i64` > A signed 64-bit integer.
- `f32` > A 32-bit float.
- `b` > An 8-bit boolean.
- `bb` > A 1-bit boolean.
- `str` > A string. Not to be confused with `String8`/`String16` in [Basic Data Structures](#basic-data-structures) below.
- `sqrt[N]` > The square root of `N`.
- `A^B` > `A` raised to the power of `B`.
- `A as B` > Value `A` as a type `B`.
- `fn bits[uint N]`
    - **return** :: Read `N` bits from the stream in the order detailed in [Reading Bits and Bytes](#reading-bits-and-bytes).
- `fn bytes[uint N]`
    - **return** :: Read `N` bytes from the stream (sequentially) in the order detailed in [Reading Bits and Bytes](#reading-bits-and-bytes).
- `fn bmc[uint COUNT, uint MAX]`
    - *data* :: Read `bits[COUNT]`.
    - *upper* :: *data* + ( 2^`COUNT` ).
    - **return** :: Return *data* if *upper* is greater than `MAX`, otherwise, if the next bit is on, return *upper*. If neither of these cases are met, return *data*.
- `cf32` > A compressed 32-bit floating point number. Read `bits[16]`, add `32768`, multiply by `32767`, then take the reciprocal.

The below core types are simply aliases of other core types and are used to differentiate between various IDs and names within the network stream.

- `ActorID<int>` > An integer (signed or unsigned) representing an actor's ID.
- `ObjectID<int>` > An integer (signed or unsigned) representing an objects's index in the objects list.
- `StreamID<int>` > An integer (signed or unsigned) representing a stream ID in the class net cache.
- `CacheID<int>` > An integer (signed or unsigned) representing a class net cache's cache ID.

## Basic Data Structures
- `String8`:
    - *length* :: Read `i32`.
    - **value** :: Read `bytes[<length>]`, drop the last byte (a null terminator), then decode with the `UTF-8` text encoding.
    > **Note**: Replay `6688` had what appears to have been a bug where one of the *length* values would be `83886080` when the actual length should have been `8`. If this value is encountered, it should be replaced with the correct value.
- `String16`:
    - *length* :: Read `i32`. If 0, return an empty string.
    - **value** :: If *length* is negative, multiply it by `-2`, then decode with the `UTF-16` text encoding. Otherwise, decode with the `Windows-1252` text encoding.
- `List[T, n=32, d=[]]`:
    - *length* :: Read \<n\> bits from the stream.
    - **values** :: If *length* is `0`, return \<d\>. Otherwise, Read \<length\> instances of `T`.
- `Rotation`:
    - **yaw** :: Read `bits[1]`. If 1, read `u8`, else null.
    - **pitch** :: Read `bits[1]`. If 1, read `u8`, else null.
    - **roll** :: Read `bits[1]`. If 1, read `u8`, else null.
- `Vector3i`:
    - *size* :: Read `bmc[4, M]` where *M* is 22 if `NET_VERSION` >= 7, else 20.
    - *bias* :: 2^(size + 1)
    - *bit_limit* :: size + 2
    - **x** :: Read `bits[<bit_limit>]`, then subtract bias.
    - **y** :: Read `bits[<bit_limit>]`, then subtract bias.
    - **z** :: Read `bits[<bit_limit>]`, then subtract bias.
- `Vector3f`:
    - *size* :: Read `bmc[4, M]` where *M* is `22` if `NET_VERSION` >= `7`, else `20`.
    - *bias* :: `2^(size + 1)`
    - *bit_limit* :: size + 2
    - **x** :: Read `bits[<bit_limit>]`, then subtract bias, then divide by 100.0.
    - **y** :: Read `bits[<bit_limit>]`, then subtract bias, then divide by 100.0.
    - **z** :: Read `bits[<bit_limit>]`, then subtract bias, then divide by 100.0.
- `Quaternion`: Deserialization differs depending on `NET_VERSION`.
    - `NET_VERSION` < `7`
        - **x** :: Read `cf32`.
        - **y** :: Read `cf32`.
        - **z** :: Read `cf32`.
        - **w** :: 0.0
    - `NET_VERSION` >= `7`
        - *largest* :: Read `bits[2]`.
        - *a* :: ( ( `bits[18]` / ( `2^18` - 1 ) ) - 0.5 ) * 2 * (1.0 / `sqrt[2]`)
        - *b* :: ( ( `bits[18]` / ( `2^18` - 1 ) ) - 0.5 ) * 2 * (1.0 / `sqrt[2]`)
        - *c* :: ( ( `bits[18]` / ( `2^18` - 1 ) ) - 0.5 ) * 2 * (1.0 / `sqrt[2]`)
        - *extra* :: `sqrt(1.0 - <a>^2 - <b>^2 - <c>^2)`
        - **x** :: *extra* if *largest* equals `0`, else *a*.
        - **y** :: *extra* if *largest* equals `1`, else *a* if *largest* equals `0`, else *b*.
        - **z** :: *extra* if *largest* equals `2`, else *b* if *largest* <= `1`, else *c*.
        - **w** :: *extra* if *largest* equals `3`, else *c*.
- `PropertySet`:
    - **value** :: Read any number of `Property` instances until a termination is signaled.
- `Property`:
    - **property_name** :: Deserialize `String8`; if result is "None", then exit early by returning nothing (or some other unique type that wont get mixed up with real values), indicating a termination.
    - **property_type** :: Deserialize `String8`.
    - **unknown_01** :: Read `u64`; does not seem to contain any useful data, and what it represents is debated.
    - **value** :: The following behavior differs depending on the **property_type**:
        - IntProperty :: Read `i32`.
        - StrProperty :: Deserialize `String16`.
        - NameProperty :: Deserialize `String16`.
        - FloatProperty :: Read `f32`.
        - ArrayProperty :: Deserialize `List[PropertySet]`.
        - ByteProperty :: A key-value pair. Deserialize one `String8` (the *key*). If the *key* is either `OnlinePlatform_Steam` or `OnlinePlatform_PS4`, the *value* is `null`. Otherwise, deserialize another `String8` (the *value*).
        - QWordProperty :: Read `u64`.
        - BoolProperty :: Read `bb`.
- `KeyFrame`:
    - **time** :: Read `f32`.
    - **frame** :: Read `u32`.
    - **file_position** :: Read `u32`.
- `DebugString`:
    - **frame** :: Read `u32`.
    - **username** :: Deserialize `String16`.
    - **text** :: Deserialize `String16`.
- `TickMark`:
    - **description** :: Deserialize `String16`.
    - **frame** :: Read `u32`.
- `Class`:
    - **class** :: Deserialize `String8`.
    - **index** :: Read `u32`.
- `ClassNetCacheProperty`:
    - **object_id** :: Read `u32`.
    - **stream_id** :: Read `u32`.
- `ClassNetCacheEntry`:
    - **object_id** :: Read `u32`.
    - **parent_id** :: Read `u32`.
    - **cache_id** :: Read `u32`. 
    - **properties** :: Deserialize `List[ClassNetCacheProperty]`

## Advanced Data Structures (Attributes)

All 40 attributes and how to deserialize them. Below you will see more than the core 40 attributes as there are some supporting data structures that aren't actually attributes in and of themselves.

- `ActiveActorAttr`:
    - **active** :: Read `bb`.
    - **actor_id** :: Read `i32 as ActorID`.
- `AppliedDamageAttr`:
    - **id** :: Read `u8`.
    - **position** :: Deserialize `Vector3f`.
    - **damage_index** :: Read `i32`.
    - **total_damage** :: Read `i32`.
- `BooleanAttr` > Read `bb`.
- `ByteAttr` > Read `u8`.
- `CameraSettingsAttr`:
    - **fov** :: Read `f32`.
    - **height** :: Read `f32`.
    - **angle** :: Read `f32`.
    - **distance** :: Read `f32`.
    - **stiffness** :: Read `f32`.
    - **swivel** :: Read `f32`.
    - **transition** :: Read `f32` if `ENGINE_VERSION` >= `868` AND `LICENSEE_VERSION` >= `20`. Otherwise, this value is `null`.
- `ClubColorsAttr`:
    - **blue_flag** :: Read `bb`.
    - **blue_color** :: Read `u8`.
    - **orange_flag** :: Read `bb`.
    - **orange_color** :: Read `u8`.
- `DamageStateAttr`:
    - **tile_state** :: Read `u8`.
    - **damaged** :: Read `bb`.
    - **offender** :: Read `i32 as ActorID`.
    - **ball_position** :: Deserialize `Vector3f`.
    - **direct_hit** :: Read `bb`.
    - **unknown_02** :: Read `bb`.
- `DemolishAttr`:
    - **attacker_flag** :: Read `bb`.
    - **attacker** :: Read `i32 as ActorID`.
    - **victim_flag** :: Read `bb`.
    - **victim** :: Read `i32 as ActorID`.
    - **attack_velocity** :: Deserialize `Vector3f`.
    - **victim_velocity** :: Deserialize `Vector3f`.
- `DemolishFXAttr`:
    - **custom_demo_flag** :: Read `bb`.
    - **custom_demo_id** :: Read `i32`.
    - **demolish** :: Deserialize `DemolishAttr`.
- `EnumAttr` > Read `bits[11]`.
- `ExplosionAttr`:
    - **flag** :: Read `bb`.
    - **actor_id** :: Read `i32 as ActorID`.
    - **location** :: Deserialize `Vector3f`.
- `ExtendedExplosionAttr`:
    - **explosion** :: Deserialize `ExplosionAttr`.
    - **unknown_03** :: Read `bb`.
    - **secondary_actor_id** :: Read `i32 as ActorID`.
- `FlaggedByteAttr`:
    - **flag** :: Read `bb`.
    - **byte** :: Deserialize `ByteAttr`.
- `FloatAttr` > Read `f32`.
- `GameModeAttr`:
    - **init** :: `8` if `ENGINE_VERSION` >= `868` AND `LICENSEE_VERSION` >= `12`, else `8`.
    - **value** :: Read `bits[<init>]`.
- `IntAttr` > Read `i32`.
- `Int64Attr` > Read `i64`.
- `LoadoutAttr`:
    - **version** :: Read `u8`.
    - **body** :: Read `u32`.
    - **decal** :: Read `u32`.
    - **wheels** :: Read `u32`.
    - **rocket_trail** :: Read `u32`.
    - **antenna** :: Read `u32`.
    - **topper** :: Read `u32`.
    - **unknown_04** :: Read `u32`.
    - **unknown_05** :: Read `u32` if **version** >= `9`, else `null`.
    - **engine_audio** :: Read `u32` if **version** >= `16`, else `null`.
    - **trail** :: Read `u32` if **version** >= `16`, else `null`.
    - **goal_explosion** :: Read `u32` if **version** >= `16`, else `null`.
    - **banner** :: Read `u32` if **version** >= `17`, else `null`.
    - **product_id** :: Read `u32` if **version** >= `19`, else `null`.
    - **unknown_06** :: Read `u32` if **version** >= `22`, else `null`.
    - **unknown_07** :: Read `u32` if **version** >= `22`, else `null`.
    - **unknown_08** :: Read `u32` if **version** >= `22`, else `null`.
- `Product`:
    - **unknown_09** :: Read `bb`.
    - **object_id** :: Read `u32`.
    - **value, type** :: Deserialization depends upon the object name (from **object_id**):
        - `TAGame.ProductAttribute_UserColor_TA` :: If `ENGINE_VERSION` >= `868` AND `LICENSEE_VERSION` >= `23` AND `NET_VERSION` >= `8`, read `i32` (**type**="NewColor"), else if the next bit is on, read `bits[31]` (**type**="OldColor"). Otherwise, `null` (**type**="NoColor").
        - `TAGame.ProductAttribute_Painted_TA` :: If `ENGINE_VERSION` >= `868` AND `LICENSEE_VERSION` >= `18`, read `bits[31]` (**type**="NewPaint"), else read `bmc[3, 14]` (**type**="OldPaint").
        - `TAGame.ProductAttribute_SpecialEdition_TA` :: Read `bits[31]` (**type**="SpecialEdition").
        - `TAGame.ProductAttribute_TeamEdition_TA` :: If `ENGINE_VERSION` >= `868` AND `LICENSEE_VERSION` >= `18`, read `bits[31]` (**type**="NewTeamEdition"), else read `bmc[3, 14]` (**type**="OldTeamEdition").
        - `TAGame.ProductAttribute_TitleID_TA` :: Deserialize `String16` (**type**="Title").
        - If the object did not match any of the above, return `null` (**type**="Absent").
- `LoadoutOnlineAttr`:
    - **values** :: Read `List[ List[ Product, n=8, d=null ], n=8, d=null ]`
- `LoadoutsOnlineAttr`:
    - **blue** :: Deserialize `LoadoutOnlineAttr`.
    - **orange** :: Deserialize `LoadoutOnlineAttr`.
    - **unknown_10** :: Read `bb`.
    - **unknown_11** :: Read `bb`.
- `LocationAttr` > Deserialize `Vector3f`.
- `MusicStingerAttr`:
    - **flag** :: Read `bb`.
    - **cue** :: Read `u32`.
    - **trigger** :: Read `u8`.
- `PartyLeaderAttr`:
    - *system_id* :: Read `u8`.
    - **unique_id** :: If *system_id* is `0`, return `null`. Otherwise, deserialize and return a `UniqueIDAttr` with a precomputed *system_id* of *system_id*.
- `PickupAttr`:
    - **instigator** :: If the next bit is on, read `i32 as ActorID`, else null.
    - **picked_up** :: Read `bb`.
- `PickupInfoAttr`:
    - **active** :: Read `bb`.
    - **actor_id** :: Read `i32 as ActorID`.
    - **items_are_preview** :: Read `bb`.
    - **unknown_12** :: Read `bb`.
    - **unknown_13** :: Read `bb`.
- `PickupNewAttr`:
    - **instigator** :: If the next bit is on, read `i32 as ActorID`, else null.
    - **picked_up** :: Read `u8`.
- `PlayerHistoryKey` > Read `bits[14] as ByteAttr`.
- `PrivateMatchSettingsAttr`:
    - **mutators** :: Deserialize `String16`.
    - **joinable_by** :: Read `u32`.
    - **max_players** :: Read `u32`.
    - **game_name** :: Deserialize `String16`.
    - **password** :: Deserialize `String16`.
    - **flag** :: Read `bb`.
- `QWordStringAttr` > Deserialize `String16` if `IS_RL_223` is `true`, else read `u64`.
- `RepStatTitleAttr`:
    - **unknown_14** :: Read `bb`.
    - **name** :: Deserialize `String16`.
    - **unknown_15** :: Read `bb`.
    - **index** :: Read `u32`.
    - **value** :: Read `u32`.
- `ReservationAttr`:
    - **number** :: Read `bits[3]`.
    - **unique_id** :: Read `UniqueIDAttr`.
    - **name** :: `null` if the *system_id* of `UniqueIDAttr` is `0`, else deserialize `String16`.
    - **unknown_16** :: Read `bb`.
    - **unknown_17** :: Read `bb`.
    - **unknown_18** :: Read `bits[6]` if `ENGINE_VERSION` >= `868` AND `LICENSEE_VERSION` >= `12`, else `null`.
- `RemoteID.Epic`:
    - **online_id** :: Deserialize `String16`.
- `RemoteID.PlayStation`:
    - **name** :: Read `bytes[16]`, filter out any `null` bytes (`0x00`), then decode with the `Windows-1252` character encoding.
    - **unknown_19** :: Read `bytes[16]` if `NET_VERSION` >= `1`, else read `bytes[8]`.
    - **online_id** :: Read `u64`.
- `RemoteID.PsyNet`:
    - **online_id** :: Read `u64`.
    - **unknown_20** :: `null` if `NET_VERSION` >= `10` else read `bytes[24]`.
- `RemoteID.QQ`:
    - **online_id** :: Read `u64`.
- `RemoteID.SplitScreen`:
    - **online_id** :: Read `bits[24]`.
- `RemoteID.Steam`:
    - **online_id** :: Read `u64`.
- `RemoteID.Switch`:
    - **online_id** :: Read `u64`.
    - **unknown_21** :: Read `bytes[24]`.
- `RemoteID.Xbox`:
    - **online_id** :: Read `u64`.
- `RigidBodyAttr`:
    - **sleeping** :: Read `bb`.
    - **location** :: Deserialize `Vector3f`.
    - **rotation** :: Deserialize `Quaternion`.
    - **linear_velocity** :: Deserialize `Vector3f` if **sleeping** is `false`, else `null`.
    - **angular_velocity** :: Deserialize `Vector3f` if **sleeping** is `false`, else `null`.
- `RotationAttr` > Deserialize `Rotation`.
- `StatEventAttr`:
    - **unknown_22** :: Read `bb`.
    - **object_id** :: Read `i32 as ObjectID`.
- `StringAttr` > Deserialize `String16`.
- `TeamLoadoutAttr`:
    - **blue** :: Deserialize `Loadout`.
    - **orange** :: Deserialize `Loadout`.
- `TeamPaintAttr`:
    - **team** :: Read `u8`.
    - **primary_color** :: Read `u8`.
    - **accent_color** :: Read `u8`.
    - **primary_finish** :: Read `u32`.
    - **accent_finish** :: Read `u32`.
- `UniqueIDAttr`:
    - **system_id** :: Read `u8`. In the case of `PartyLeaderAttr`, the provided **system_id** should be used instead of reading from the stream here.
    - **remote_id** :: Deserialize a different remote ID depending on the value of **system_id**:
        - `0` :: Deserialize `RemoteID.SplitScreen`.
        - `1` :: Deserialize `RemoteID.Steam`.
        - `2` :: Deserialize `RemoteID.PlayStation`.
        - `4` :: Deserialize `RemoteID.Xbox`.
        - `5` :: Deserialize `RemoteID.QQ`.
        - `6` :: Deserialize `RemoteID.Switch`.
        - `7` :: Deserialize `RemoteID.PsyNet`.
        - `11` :: Deserialize `RemoteID.Epic`.
    - **local_id** :: Read `u8`.
- `TitleAttr`:
    - **unknown_23** :: Read `bb`.
    - **unknown_24** :: Read `bb`.
    - **unknown_25** :: Read `u32`.
    - **unknown_26** :: Read `u32`.
    - **unknown_27** :: Read `u32`.
    - **unknown_28** :: Read `u32`.
    - **unknown_29** :: Read `u32`.
    - **unknown_30** :: Read `bb`.
- `WeldedAttr`:
    - **active** :: Read `bb`.
    - **actor_id** :: Read `i32 as ActorID`.
    - **offset** :: Deserialize `Vector3f`.
    - **mass** :: Read `f32`.
    - **rotation** :: Deserialize `Rotation`.

# Replay Deserialization - Basic Data

Now that we know how to deserialize all of the different types and data structures available to us, we can begin to deserialize the replay. All Rocket League replays consist of three main sections:

1. The header, which contains some top-level information about the replay as a whole.
2. The body, which contains some mid-level information, as well as the network stream, which is essentially the a raw download of the network stream sent from the Psyonix servers to the client during the match.
3. The footer, which contains some lower-level information about the replay, most of which is required to deserialize the network stream.

Now, lets look at each section in detail. Note that these sections should be read in sequence, in the same order as given below.

## Deserializing the Header

1. `u32` HeaderLength :: The length (in bytes) of the header section.
2. `u32` HeaderCRC :: The cyclic redundancy check for the header section.
3. `u32` EngineVersion :: The engine version. This is the `ENGINE_VERSION` variable mentioned in [Types and Structures](#types-and-structures).
4. `u32` LicenseeVersion :: The licensee version. This is the `LICENSEE_VERSION` variable mentioned in [Types and Structures](#types-and-structures).
5. `u32` NetVersion :: The net version. ONLY present if `ENGINE_VERSION` >= `866` and `LICENSEE_VERSION` >= `18`. This is the `NET_VERSION` variable mentioned in [Types and Structures](#types-and-structures), and is usually set to `0` if not present in the replay in order to make comparisons easier.
6. `String16` VersionID :: The version ID.
7. `PropertySet` HeaderProperties :: Basic game info.

## Deserializing the Body

1. `u32` BodyLength :: The length of the body + footer.
2. `u32` BodyCRC :: The cyclic redundancy check for the body + footer.
3. `List[String16]` Levels :: A list of `String16`s detailing SFX packages.
4. `List[KeyFrame]` KeyFrames :: Keyframes in the replay.
5. `u32` NetworkStreamLength :: The length (in bytes) of the body (network stream).
6. `bytes[<NetworkStreamLength>]` NetworkStream :: The entire network stream. Deserializing the stream is by far the most difficult part of the replay deserialization process, and in order to do it, we actually need to skip past the stream and read the footer first. You can read the bytes, or simply seek past it.

## Deserializing the Footer

1. `List[DebugString]` DebugStrings :: Debug information.
2. `List[TickMark]` TickMarks :: The frames in which a goal was scored or a save was made.
3. `List[String16]` Packages :: Various packages from CookedPCConsole that were used in the game.
4. `List[String16]` Objects :: A list of objects used in the game.
5. `List[String16]` Names :: A list of object names used in the game.
6. `List[Class]` Classes :: A list of class indicies.
7. `List[ClassNetCacheEntry]` ClassNetCache :: A list of network attribute encodings.

# Replay Deserialization - Network Stream Preparation

Deserializing the network stream is significantly more difficult than the other basic information that is in the replay. But before we even start deserializing the network stream, we need to prepare some data beforehand. First, let's look at the structure of the class net cache.

## Preparation of the Class Net Cache

Here is the example class net cache we will be working with:

```json
[
    {
        "object_id": 40,
        "parent_id": 20,
        "cache_id": 38,
        "properties": [
            {
                "object_id": 22,
                "stream_id": 20
            }
        ]
    },
    {
        "object_id": 41,
        "parent_id": 38,
        "cache_id": 38,
        "properties": []
    },
    {
        "object_id": 52,
        "parent_id": 38,
        "cache_id": 48,
        "properties": [
            {
                "object_id": 42,
                "stream_id": 38
            }
        ]
    }
]
```

This is just an example - your class net cache will be much, much larger. Each entry is made up of an `object_id`, which points to that entry's parent class object in the `Objects` list (from the footer). The `cache_id` is a (not-so-) unique id that identifies a given entry for inter-entry relationships. The `parent_id` points to the `cache_id` of its parent class net cache entry. Finally, there are the `properties`, which is a list of `StreamID:ObjectID` pairs. We will need these in order to know how many bits to read when accumulating attributes for an updated actor, as well as being able to decode the recently read stream ID into the correct object ID, and eventually into the corresponding attribute type. But first, lets make some changes to the class net cache, as in its raw state, it is actually quite condensed and can often contain numerous hierarchical errors.

In order to get our class net cache to a usable state, we need each entry to inherit the properties of its parent entry (which may also have a parent, so this needs to be done iteratively), as well as check for incorrect `parent_id`s along the way. Something very important to note is that the `parent_id` **only** refers to the **closest** entry that appeared **before** the working entry. This is why multiple entries can have the same `cache_id` without causing issues. This process will look something like the following:

1. Create an empty class net cache for us to add updated entries to.
2. Iterate through all the class net cache entries.
3. For each entry in the original class net cache:
    1. Copy the properties into a new (temporary) list.
    2. Get the class object name from `Classes` given the entry's `object_id` (i.e. get the `class` from the item in `Classes` whose `index` value corresponds to `object_id`).
    3. If the class object name is present in the [`Class:ParentClass`](#classparentclass) hash:
        1. Plug the class object name into the hash map to get the parent class name.
        2. Get the `ObjectID` of the class object name from `Objects`.
        3. Iterate over our updated class net cache in reverse order until an entry with an `object_id` that matches the one from `3.3.2` is found. If found, add the entry's properties to our temporary properties list and break from the loop.
    4. If the class object name is *not* present in the hash, or if no parent entry was found in the iteration process explained in `3.3.3`:
        1. Iterate over our updated class net cache in reverse order until an entry with a `cache_id` is found that matches the working entry's `parent_id`. If found, add the entry's properties to our temporary properties list and break from the loop.
    5. Add a copy of the working entry into our updated class net cache, but replace its properties with our temporary properties list.

## Preparation of Miscellaneous Information

There is also some miscellaneous (and luckily easy to calculate) information that will be useful to have pre-computed before we begin deserializing the network stream:

1. `IS_RL_223`: `true` if the `BuildVersion` property is present in the header properties AND its value >= `221120.42953.406184`, else `false`.
2. `IS_LAN`: `true` if the value of the `MatchType` header property is `Lan`, else `false`.
3. `PARSE_ACTOR_NAME_ID`: `true` if (`ENGINE_VERSION` >= `868` AND `LICENSEE_VERSION` >= `20`) OR (`ENGINE_VERSION` >= `868` AND `LICENSEE_VERSION` >= `14` AND `IS_LAN` is `false`), else `false`.
4. `ACTOR_ID_MAX`: The value of the `MaxChannels` header property. If not present, then use `1023`.
5. `ACTOR_ID_SIZE`: One less than the bit length of `ACTOR_ID_MAX`, or 0; whichever is larger.

Something else we will need is an empty hash (which we will call `ACTIVE_ACTORS`) that will be dynamically updated as new actors get added and removed (a process which will be covered in detail in the next section). This will be an `ActorID:ObjectID` hash.

# Replay Deserialization - Network Stream

Finally, its time to start deserializing the network stream. The network stream is formatted as a list of frames, the length of which is hinted at in the `NumFrames` header property. If the `NumFrames` header property is not present, then no frames should be deserialized. Each frame starts with two 32-bit floating point numbers - the absolute time (in seconds) and the number of seconds since the previous frame, respectively. Next comes the actor section. This section consists of zero or more actor segments, each of which can perform one of three tasks - create a new actor, update an existing actor, or delete an actor. This process is actually quite simple:

- While `bb` == `true`:
    - `actor_id` = `bmc[C, M] as ActorID` where `C` is `ACTOR_ID_SIZE` and `M` is `ACTOR_ID_MAX`, both of which were prepared beforehand (see [`Preparation of Miscellaneous Information`](#preparation-of-miscellaneous-information)).
    - If `bb` == `true` (actor is alive):
        - If `bb` == `true` (actor is new):
            - Deserialize a new actor.
        - Else (actor is updated):
            - Update the actor (given `actor_id`).
    - Else (actor destroyed): 
        - Allow `actor_id` to be reused.
- Else:
    - End of frame.

Each frame should include a list of new, updated, and deleted actors that will be dynamically filled as your program proceeds through the deserialization process.

## Deserializing a New Actor

Deserializing a new actor is pretty simple:

- **name_id** :: `null` if `PARSE_ACTOR_NAME_ID` is `false`, else, read `i32`.
- **unknown_31** :: Read `bb`.
- **object_id** :: Read `i32`.
- *object_name* :: Get the object name from `Objects` (from the footer) given **object_id**.
- *spawn_trajectory* :: Get the value from the [`Object:SpawnTrajectory`](#objectspawntrajectory) hash given *object_name* as the key, or if not present in the hash, use `[false, false]` instead.
- **initial_position** :: If the first item in *spawn_trajectory* is `true`, then read `Vector3i`. Else `null`.
- **initial_rotation** :: If the second item in *spawn_trajectory* is `true`, then read `Rotation`. Else `null`.

That's all the data for a new actor. As mentioned briefly in [`Preparation of Miscellaneous Information`](#preparation-of-miscellaneous-information), we also need to add an entry to the `ACTIVE_ACTORS` hash (given the `actor_id` and **object_id**). Additionally, we should append this actor to the list of new actors in the current frame.

## Deserializing a Deleted Actor

Deleted actors don't actually come with any data, other than the `actor_id` decoded in the frame's actor segment loop. All we need to do is remove the corresponding entry from `ACTIVE_ACTORS` and append the actor ID to the frame's list of deleted actors.

## Deserializing an Updated Actor

Updating an existing actor is the most difficult of the three operations. Here is the general flow:

- *object_id* :: Acquire the value from `ACTIVE_ACTORS` given `actor_id`.
- *object_name* :: Acquire the object name from the `Objects` list given our *object_id*.
- *parent_object_name* :: Acquire the parent object's name given *object_name*. The method for achieving this will be explained later in this section.
- *parent_object_id* :: The index of *parent_object_name* in the `Objects` list.
- *class_net_cache_entry* :: Find the entry from our updated class net cache whose `object_id` matches *parent_object_id*.
- *max_stream_id* :: If the *class_net_cache_entry* has no properties, then this is `3`. Otherwise, get the maximum `stream_id` value from the *class_net_cache_entry*'s properties, then add `1`.
- *stream_id_size* :: Get the bit length of *max_stream_id*, then subtract `1`.
- **attributes** :: Now we read the attributes. While the next bit is on (i.e. a loop `while bits[1]`), do the following:
    - **stream_id** :: Read `bmc[C, M]` where `C` is *stream_id_size* and `M` is *max_stream_id*.
    - **object_id** :: Find the property in *class_net_cache_entry*'s properties given **stream_id**, and get the corresponding `object_id`.
    - **attribute_type** :: Get the attribute type from the [`Object:AttributeType`](#objectattributetype) hash given the object name that corresponds to **object_id**.
        > **Note**: In order to increase efficiency, the **object_id** and **attribute_type** should be incorperated directly into the updated class net cache, as detailed in [`Preparation of the Class Net Cache`](#preparation-of-the-class-net-cache). Additionally, the *max_stream_id* and *stream_id_size* should ideally only be calculated once, most likely during the preprocess where the class net cache is updated.

There is one part of the process that still needs to be covered; that is, a way to find the parent object's name given a child object. This is an incredibly important step, as all of the entries in the class net cache refer to the parent object, while new actors are often created as children of those parent objects. To find the parent object, we can plug the child object's name into the [`Object:Parent`](#objectparent) hash to get the parent object's name. If the object's name is not present in the hash, then we need to do the following:

- If `TheWorld:PersistentLevel.CrowdActor_TA` is a substring of the object name, then the parent object is `TheWorld:PersistentLevel.CrowdActor_TA`.
- If `TheWorld:PersistentLevel.VehiclePickup_Boost_TA` is a substring of the object name, then the parent object is `TheWorld:PersistentLevel.VehiclePickup_Boost_TA`.
- If `TheWorld:PersistentLevel.CrowdManager_TA` is a substring of the object name, then the parent object is `TheWorld:PersistentLevel.CrowdManager_TA`.
- If `TheWorld:PersistentLevel.BreakOutActor_Platform_TA` is a substring of the object name, then the parent object is `TheWorld:PersistentLevel.BreakOutActor_Platform_TA`.
- If `TheWorld:PersistentLevel.InMapScoreboard_TA` is a substring of the object name, then the parent object is `TheWorld:PersistentLevel.InMapScoreboard_TA`.
- If `TheWorld:PersistentLevel.HauntedBallTrapTrigger_TA` is a substring of the object name, then the parent object is `TheWorld:PersistentLevel.HauntedBallTrapTrigger_TA`.
- If `:GameReplicationInfoArchetype` is a substring of the object name, then the parent object is `TAGame.GRI_TA`.

This can be pretty taxing, given that there are often 100,000 or more updated actors in a single replay. For this reason, we can significantly improve throughput by dynamically adding a copy of the updated class net cache entry to the updated class net cache with the **object_id** of our updated actor as the key, so that next time we can simply pull from the class net cache without having to go through the process of converting object ID to object name, object name to parent object name, and parent object name to parent object id.

# Hash Maps

## Object:SpawnTrajectory

Note: `SpawnTrajectory` is given as [\<has initial location\>, \<has initial rotation\>]

```json
{
    "TAGame.Ball_Breakout_TA": [true, true],
    "Archetypes.Ball.Ball_Breakout": [true, true],
    "Archetypes.Ball.Ball_Trajectory": [true, true],
    "TAGame.Ball_TA": [true, true],
    "Archetypes.Ball.Ball_BasketBall_Mutator": [true, true],
    "Archetypes.Ball.Ball_BasketBall": [true, true],
    "Archetypes.Ball.Ball_Basketball": [true, true],
    "Archetypes.Ball.Ball_Default": [true, true],
    "Archetypes.Ball.Ball_God": [true, true],
    "Archetypes.Ball.Ball_Puck": [true, true],
    "Archetypes.Ball.Ball_Anniversary": [true, true],
    "Archetypes.Ball.CubeBall": [true, true],
    "Archetypes.Ball.Ball_Haunted": [true, true],
    "Archetypes.Ball.Ball_Training": [true, true],
    "Archetypes.Ball.Ball_Football": [true, true],
    "TAGame.Ball_Haunted_TA": [true, true],
    "TAGame.Car_Season_TA": [true, true],
    "TAGame.Car_TA": [true, true],
    "Archetypes.Car.Car_Default": [true, true],
    "Archetypes.GameEvent.GameEvent_Season:CarArchetype": [true, true],
    "Archetypes.SpecialPickups.SpecialPickup_HauntedBallBeam": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_Rugby": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_Football": [true, false],
    "TAGame.CameraSettingsActor_TA": [true, false],
    "TAGame.CarComponent_Boost_TA": [true, false],
    "TAGame.CarComponent_Dodge_TA": [true, false],
    "TAGame.CarComponent_DoubleJump_TA": [true, false],
    "TAGame.CarComponent_DoubleJump_TA:DoubleJumpImpulse": [true, false],
    "TAGame.CarComponent_FlipCar_TA": [true, false],
    "TAGame.CarComponent_Jump_TA": [true, false],
    "TAGame.GameEvent_Season_TA": [true, false],
    "TAGame.GameEvent_Soccar_TA": [true, false],
    "TAGame.GameEvent_SoccarPrivate_TA": [true, false],
    "TAGame.GameEvent_SoccarSplitscreen_TA": [true, false],
    "TAGame.GRI_TA": [true, false],
    "TAGame.PRI_TA": [true, false],
    "TAGame.SpecialPickup_BallCarSpring_TA": [true, false],
    "TAGame.SpecialPickup_BallFreeze_TA": [true, false],
    "TAGame.SpecialPickup_BallGravity_TA": [true, false],
    "TAGame.SpecialPickup_BallLasso_TA": [true, false],
    "TAGame.SpecialPickup_BallVelcro_TA": [true, false],
    "TAGame.SpecialPickup_Batarang_TA": [true, false],
    "TAGame.SpecialPickup_BoostOverride_TA": [true, false],
    "TAGame.SpecialPickup_GrapplingHook_TA": [true, false],
    "TAGame.SpecialPickup_HitForce_TA": [true, false],
    "TAGame.SpecialPickup_Swapper_TA": [true, false],
    "TAGame.SpecialPickup_Tornado_TA": [true, false],
    "TAGame.Team_Soccar_TA": [true, false],
    "Archetypes.CarComponents.CarComponent_Boost": [true, false],
    "Archetypes.CarComponents.CarComponent_Dodge": [true, false],
    "Archetypes.CarComponents.CarComponent_DoubleJump": [true, false],
    "Archetypes.CarComponents.CarComponent_FlipCar": [true, false],
    "Archetypes.CarComponents.CarComponent_Jump": [true, false],
    "Archetypes.GameEvent.GameEvent_Basketball": [true, false],
    "Archetypes.GameEvent.GameEvent_BasketballPrivate": [true, false],
    "Archetypes.GameEvent.GameEvent_BasketballSplitscreen": [true, false],
    "Archetypes.GameEvent.GameEvent_Breakout": [true, false],
    "Archetypes.GameEvent.GameEvent_Hockey": [true, false],
    "Archetypes.GameEvent.GameEvent_HockeyPrivate": [true, false],
    "Archetypes.GameEvent.GameEvent_HockeySplitscreen": [true, false],
    "Archetypes.GameEvent.GameEvent_Items": [true, false],
    "Archetypes.GameEvent.GameEvent_Season": [true, false],
    "Archetypes.GameEvent.GameEvent_Soccar": [true, false],
    "Archetypes.GameEvent.GameEvent_SoccarLan": [true, false],
    "Archetypes.GameEvent.GameEvent_SoccarPrivate": [true, false],
    "Archetypes.GameEvent.GameEvent_SoccarSplitscreen": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_BallFreeze": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_BallGrapplingHook": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_BallLasso": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_BallSpring": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_BallVelcro": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_Batarang": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_BoostOverride": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_CarSpring": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_GravityWell": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_StrongHit": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_Swapper": [true, false],
    "Archetypes.SpecialPickups.SpecialPickup_Tornado": [true, false],
    "Archetypes.Teams.Team0": [true, false],
    "Archetypes.Teams.Team1": [true, false],
    "GameInfo_Basketball.GameInfo.GameInfo_Basketball:GameReplicationInfoArchetype": [true, false],
    "GameInfo_Breakout.GameInfo.GameInfo_Breakout:GameReplicationInfoArchetype": [true, false],
    "Gameinfo_Hockey.GameInfo.Gameinfo_Hockey:GameReplicationInfoArchetype": [true, false],
    "GameInfo_Items.GameInfo.GameInfo_Items:GameReplicationInfoArchetype": [true, false],
    "GameInfo_Season.GameInfo.GameInfo_Season:GameReplicationInfoArchetype": [true, false],
    "GameInfo_Soccar.GameInfo.GameInfo_Soccar:GameReplicationInfoArchetype": [true, false],
    "GameInfo_Tutorial.GameInfo.GameInfo_Tutorial:GameReplicationInfoArchetype": [true, false],
    "GameInfo_FootBall.GameInfo.GameInfo_FootBall:Archetype": [true, false],
    "GameInfo_FootBall.GameInfo.GameInfo_FootBall:GameReplicationInfoArchetype": [true, false],
    "TAGame.Default__CameraSettingsActor_TA": [true, false],
    "TAGame.Default__PRI_TA": [true, false],
    "TAGame.Default__PRI_Breakout_TA": [true, false],
    "TAGame.Default__MaxTimeWarningData_TA": [true, false],
    "TAGame.Default__RumblePickups_TA": [true, false],
    "TAGame.Default__PickupTimer_TA": [true, false],
    "TheWorld:PersistentLevel.BreakOutActor_Platform_TA": [true, false],
    "TheWorld:PersistentLevel.CrowdActor_TA": [true, false],
    "TheWorld:PersistentLevel.CrowdManager_TA": [true, false],
    "TheWorld:PersistentLevel.InMapScoreboard_TA": [true, false],
    "TheWorld:PersistentLevel.VehiclePickup_Boost_TA": [true, false],
    "TAGame.HauntedBallTrapTrigger_TA": [true, false],
    "ProjectX.Default__NetModeReplicator_X": [true, false],
    "GameInfo_Tutorial.GameEvent.GameEvent_Tutorial_Aerial": [true, false],
    "Archetypes.Tutorial.Cannon": [true, false],
    "gameinfo_godball.GameInfo.gameinfo_godball:Archetype": [true, false],
    "gameinfo_godball.GameInfo.gameinfo_godball:GameReplicationInfoArchetype": [true, false],
}
```

## Object:AttributeType

```json
{
    "Engine.Actor:bBlockActors": "BooleanAttr",
    "Engine.Actor:bCollideActors": "BooleanAttr",
    "Engine.Actor:bHidden": "BooleanAttr",
    "Engine.Actor:bTearOff": "BooleanAttr",
    "Engine.Actor:DrawScale": "FloatAttr",
    "Engine.Actor:RemoteRole": "EnumAttr",
    "Engine.Actor:Role": "EnumAttr",
    "Engine.Actor:Rotation": "RotationTagAttr",
    "Engine.GameReplicationInfo:bMatchIsOver": "BooleanAttr",
    "Engine.GameReplicationInfo:GameClass": "ActiveActorAttr",
    "Engine.GameReplicationInfo:ServerName": "StringAttr",
    "Engine.Pawn:PlayerReplicationInfo": "ActiveActorAttr",
    "Engine.Pawn:HealthMax": "IntAttr",
    "Engine.PlayerReplicationInfo:bBot": "BooleanAttr",
    "Engine.PlayerReplicationInfo:bIsSpectator": "BooleanAttr",
    "Engine.PlayerReplicationInfo:bReadyToPlay": "BooleanAttr",
    "Engine.PlayerReplicationInfo:bTimedOut": "BooleanAttr",
    "Engine.PlayerReplicationInfo:bWaitingPlayer": "BooleanAttr",
    "Engine.PlayerReplicationInfo:Ping": "ByteAttr",
    "Engine.PlayerReplicationInfo:PlayerID": "IntAttr",
    "Engine.PlayerReplicationInfo:PlayerName": "StringAttr",
    "Engine.PlayerReplicationInfo:RemoteUserData": "StringAttr",
    "Engine.PlayerReplicationInfo:Score": "IntAttr",
    "Engine.PlayerReplicationInfo:Team": "ActiveActorAttr",
    "Engine.PlayerReplicationInfo:UniqueId": "UniqueIdAttr",
    "Engine.TeamInfo:Score": "IntAttr",
    "Engine.ReplicatedActor_ORS:ReplicatedOwner": "ActiveActorAttr",
    "ProjectX.GRI_X:bGameStarted": "BooleanAttr",
    "ProjectX.GRI_X:GameServerID": "QWordStringAttr",
    "ProjectX.GRI_X:MatchGUID": "StringAttr",
    "ProjectX.GRI_X:MatchGuid": "StringAttr",
    "ProjectX.GRI_X:ReplicatedGameMutatorIndex": "IntAttr",
    "ProjectX.GRI_X:ReplicatedGamePlaylist": "IntAttr",
    "ProjectX.GRI_X:ReplicatedServerRegion": "StringAttr",
    "ProjectX.GRI_X:Reservations": "ReservationAttr",
    "TAGame.Ball_Breakout_TA:AppliedDamage": "AppliedDamageAttr",
    "TAGame.Ball_Breakout_TA:DamageIndex": "IntAttr",
    "TAGame.Ball_Breakout_TA:LastTeamTouch": "ByteAttr",
    "TAGame.Ball_TA:GameEvent": "ActiveActorAttr",
    "TAGame.Ball_TA:HitTeamNum": "ByteAttr",
    "TAGame.Ball_TA:ReplicatedAddedCarBounceScale": "FloatAttr",
    "TAGame.Ball_TA:ReplicatedBallMaxLinearSpeedScale": "FloatAttr",
    "TAGame.Ball_TA:ReplicatedBallScale": "FloatAttr",
    "TAGame.Ball_TA:ReplicatedExplosionData": "ExplosionAttr",
    "TAGame.Ball_TA:ReplicatedExplosionDataExtended": "ExtendedExplosionAttr",
    "TAGame.Ball_TA:ReplicatedWorldBounceScale": "FloatAttr",
    "TAGame.Ball_God_TA:TargetSpeed": "FloatAttr",
    "TAGame.BreakOutActor_Platform_TA:DamageState": "DamageStateAttr",
    "TAGame.CameraSettingsActor_TA:bUsingBehindView": "BooleanAttr",
    "TAGame.CameraSettingsActor_TA:bMouseCameraToggleEnabled": "BooleanAttr",
    "TAGame.CameraSettingsActor_TA:bUsingSecondaryCamera": "BooleanAttr",
    "TAGame.CameraSettingsActor_TA:bUsingSwivel": "BooleanAttr",
    "TAGame.CameraSettingsActor_TA:CameraPitch": "ByteAttr",
    "TAGame.CameraSettingsActor_TA:CameraYaw": "ByteAttr",
    "TAGame.CameraSettingsActor_TA:PRI": "ActiveActorAttr",
    "TAGame.CameraSettingsActor_TA:ProfileSettings": "CamSettingsAttr",
    "TAGame.Car_TA:AddedBallForceMultiplier": "FloatAttr",
    "TAGame.Car_TA:AddedCarForceMultiplier": "FloatAttr",
    "TAGame.Car_TA:AttachedPickup": "ActiveActorAttr",
    "TAGame.Car_TA:ClubColors": "ClubColorsAttr",
    "TAGame.Car_TA:ReplicatedCarScale": "FloatAttr",
    "TAGame.Car_TA:ReplicatedDemolish": "DemolishAttr",
    "TAGame.Car_TA:ReplicatedDemolish_CustomFX": "DemolishFxAttr",
    "TAGame.Car_TA:ReplicatedDemolishGoalExplosion": "DemolishFxAttr",
    "TAGame.Car_TA:RumblePickups": "ActiveActorAttr",
    "TAGame.RumblePickups_TA:ConcurrentItemCount": "IntAttr",
    "TAGame.RumblePickups_TA:AttachedPickup": "ActiveActorAttr",
    "TAGame.RumblePickups_TA:PickupInfo": "PickupInfoAttr",
    "TAGame.Car_TA:TeamPaint": "TeamPaintAttr",
    "TAGame.CarComponent_Boost_TA:bNoBoost": "BooleanAttr",
    "TAGame.CarComponent_Boost_TA:BoostModifier": "FloatAttr",
    "TAGame.CarComponent_Boost_TA:bUnlimitedBoost": "BooleanAttr",
    "TAGame.CarComponent_Boost_TA:RechargeDelay": "FloatAttr",
    "TAGame.CarComponent_Boost_TA:RechargeRate": "FloatAttr",
    "TAGame.CarComponent_Boost_TA:ReplicatedBoostAmount": "ByteAttr",
    "TAGame.CarComponent_Boost_TA:UnlimitedBoostRefCount": "IntAttr",
    "TAGame.CarComponent_Dodge_TA:DodgeTorque": "LocationAttr",
    "TAGame.CarComponent_Dodge_TA:DodgeImpulse": "LocationAttr",
    "TAGame.CarComponent_FlipCar_TA:bFlipRight": "BooleanAttr",
    "TAGame.CarComponent_FlipCar_TA:FlipCarTime": "FloatAttr",
    "TAGame.CarComponent_TA:ReplicatedActive": "ByteAttr",
    "TAGame.CarComponent_TA:ReplicatedActivityTime": "FloatAttr",
    "TAGame.CarComponent_TA:Vehicle": "ActiveActorAttr",
    "TAGame.CrowdActor_TA:GameEvent": "ActiveActorAttr",
    "TAGame.CrowdActor_TA:ModifiedNoise": "FloatAttr",
    "TAGame.CrowdActor_TA:ReplicatedCountDownNumber": "IntAttr",
    "TAGame.CrowdActor_TA:ReplicatedOneShotSound": "ActiveActorAttr",
    "TAGame.CrowdActor_TA:ReplicatedRoundCountDownNumber": "IntAttr",
    "TAGame.CrowdManager_TA:GameEvent": "ActiveActorAttr",
    "TAGame.CrowdManager_TA:ReplicatedGlobalOneShotSound": "ActiveActorAttr",
    "TAGame.GameEvent_Soccar_TA:bBallHasBeenHit": "BooleanAttr",
    "TAGame.GameEvent_Soccar_TA:bClubMatch": "BooleanAttr",
    "TAGame.GameEvent_Soccar_TA:bOverTime": "BooleanAttr",
    "TAGame.GameEvent_Soccar_TA:bMatchEnded": "BooleanAttr",
    "TAGame.GameEvent_Soccar_TA:bNoContest": "BooleanAttr",
    "TAGame.GameEvent_Soccar_TA:bUnlimitedTime": "BooleanAttr",
    "TAGame.GameEvent_Soccar_TA:GameTime": "IntAttr",
    "TAGame.GameEvent_Soccar_TA:GameWinner": "ActiveActorAttr",
    "TAGame.GameEvent_Soccar_TA:MatchWinner": "ActiveActorAttr",
    "TAGame.GameEvent_Soccar_TA:MaxScore": "IntAttr",
    "TAGame.GameEvent_Soccar_TA:MVP": "ActiveActorAttr",
    "TAGame.GameEvent_Soccar_TA:ReplicatedMusicStinger": "MusicStingerAttr",
    "TAGame.GameEvent_Soccar_TA:ReplicatedScoredOnTeam": "ByteAttr",
    "TAGame.GameEvent_Soccar_TA:ReplicatedServerPerformanceState": "ByteAttr",
    "TAGame.GameEvent_Soccar_TA:ReplicatedStatEvent": "StatEventAttr",
    "TAGame.GameEvent_Soccar_TA:RoundNum": "IntAttr",
    "TAGame.GameEvent_Soccar_TA:SecondsRemaining": "IntAttr",
    "TAGame.GameEvent_Soccar_TA:SeriesLength": "IntAttr",
    "TAGame.GameEvent_Soccar_TA:SubRulesArchetype": "ActiveActorAttr",
    "TAGame.GameEvent_SoccarPrivate_TA:MatchSettings": "PrivateMatchSettingsAttr",
    "TAGame.GameEvent_TA:bAllowReadyUp": "BooleanAttr",
    "TAGame.GameEvent_TA:bCanVoteToForfeit": "BooleanAttr",
    "TAGame.GameEvent_TA:bHasLeaveMatchPenalty": "BooleanAttr",
    "TAGame.GameEvent_TA:BotSkill": "IntAttr",
    "TAGame.GameEvent_TA:GameMode": "GameModeAttr",
    "TAGame.GameEvent_TA:MatchTypeClass": "ActiveActorAttr",
    "TAGame.GameEvent_TA:ReplicatedGameStateTimeRemaining": "IntAttr",
    "TAGame.GameEvent_TA:ReplicatedRoundCountDownNumber": "IntAttr",
    "TAGame.GameEvent_TA:ReplicatedStateIndex": "ByteAttr",
    "TAGame.GameEvent_TA:ReplicatedStateName": "IntAttr",
    "TAGame.GameEvent_Team_TA:bForfeit": "BooleanAttr",
    "TAGame.GameEvent_Team_TA:MaxTeamSize": "IntAttr",
    "TAGame.MaxTimeWarningData_TA:EndGameWarningEpochTime": "Int64Attr",
    "TAGame.MaxTimeWarningData_TA:EndGameEpochTime": "Int64Attr",
    "TAGame.GRI_TA:NewDedicatedServerIP": "StringAttr",
    "TAGame.PRI_TA:bIsDistracted": "BooleanAttr",
    "TAGame.PRI_TA:bIsInSplitScreen": "BooleanAttr",
    "TAGame.PRI_TA:bMatchMVP": "BooleanAttr",
    "TAGame.PRI_TA:bOnlineLoadoutSet": "BooleanAttr",
    "TAGame.PRI_TA:bOnlineLoadoutsSet": "BooleanAttr",
    "TAGame.PRI_TA:BotProductName": "IntAttr",
    "TAGame.PRI_TA:bReady": "BooleanAttr",
    "TAGame.PRI_TA:bUsingBehindView": "BooleanAttr",
    "TAGame.PRI_TA:bUsingItems": "BooleanAttr",
    "TAGame.PRI_TA:bUsingSecondaryCamera": "BooleanAttr",
    "TAGame.PRI_TA:CameraPitch": "ByteAttr",
    "TAGame.PRI_TA:CameraSettings": "CamSettingsAttr",
    "TAGame.PRI_TA:CameraYaw": "ByteAttr",
    "TAGame.PRI_TA:ClientLoadout": "LoadoutAttr",
    "TAGame.PRI_TA:ClientLoadoutOnline": "LoadoutOnlineAttr",
    "TAGame.PRI_TA:ClientLoadouts": "TeamLoadoutAttr",
    "TAGame.PRI_TA:ClientLoadoutsOnline": "LoadoutsOnlineAttr",
    "TAGame.PRI_TA:ClubID": "Int64Attr",
    "TAGame.PRI_TA:MatchAssists": "IntAttr",
    "TAGame.PRI_TA:MatchBreakoutDamage": "IntAttr",
    "TAGame.PRI_TA:MatchGoals": "IntAttr",
    "TAGame.PRI_TA:MatchSaves": "IntAttr",
    "TAGame.PRI_TA:MatchScore": "IntAttr",
    "TAGame.PRI_TA:MatchShots": "IntAttr",
    "TAGame.PRI_TA:MaxTimeTillItem": "IntAttr",
    "TAGame.PRI_TA:PartyLeader": "PartyLeaderAttr",
    "TAGame.PRI_TA:PawnType": "ByteAttr",
    "TAGame.PRI_TA:PersistentCamera": "ActiveActorAttr",
    "TAGame.PRI_TA:PlayerHistoryKey": "PlayerHistoryKeyAttr",
    "TAGame.PRI_TA:PlayerHistoryValid": "BooleanAttr",
    "TAGame.PRI_TA:ReplicatedGameEvent": "ActiveActorAttr",
    "TAGame.PRI_TA:ReplicatedWorstNetQualityBeyondLatency": "ByteAttr",
    "TAGame.PRI_TA:RepStatTitles": "RepStatTitleAttr",
    "TAGame.PRI_TA:SteeringSensitivity": "FloatAttr",
    "TAGame.PRI_TA:SkillTier": "FlaggedByteAttr",
    "TAGame.PRI_TA:TimeTillItem": "IntAttr",
    "TAGame.PRI_TA:Title": "IntAttr",
    "TAGame.PRI_TA:TotalXP": "IntAttr",
    "TAGame.PRI_TA:PrimaryTitle": "TitleAttr",
    "TAGame.PRI_TA:SecondaryTitle": "TitleAttr",
    "TAGame.PRI_TA:SpectatorShortcut": "IntAttr",
    "TAGame.PRI_TA:CurrentVoiceRoom": "StringAttr",
    "TAGame.RBActor_TA:bFrozen": "BooleanAttr",
    "TAGame.RBActor_TA:bIgnoreSyncing": "BooleanAttr",
    "TAGame.RBActor_TA:bReplayActor": "BooleanAttr",
    "TAGame.RBActor_TA:ReplicatedRBState": "RigidBodyAttr",
    "TAGame.RBActor_TA:WeldedInfo": "WeldedAttr",
    "TAGame.SpecialPickup_BallFreeze_TA:RepOrigSpeed": "FloatAttr",
    "TAGame.SpecialPickup_BallVelcro_TA:AttachTime": "FloatAttr",
    "TAGame.SpecialPickup_BallVelcro_TA:bBroken": "BooleanAttr",
    "TAGame.SpecialPickup_BallVelcro_TA:bHit": "BooleanAttr",
    "TAGame.SpecialPickup_BallVelcro_TA:BreakTime": "FloatAttr",
    "TAGame.SpecialPickup_Targeted_TA:Targeted": "ActiveActorAttr",
    "TAGame.SpecialPickup_Football_TA:WeldedBall": "ActiveActorAttr",
    "TAGame.Team_Soccar_TA:GameScore": "IntAttr",
    "TAGame.Team_TA:ClubColors": "ClubColorsAttr",
    "TAGame.Team_TA:ClubID": "Int64Attr",
    "TAGame.Team_TA:CustomTeamName": "StringAttr",
    "TAGame.Team_TA:Difficulty": "IntAttr",
    "TAGame.Team_TA:GameEvent": "ActiveActorAttr",
    "TAGame.Team_TA:LogoData": "ActiveActorAttr",
    "TAGame.Vehicle_TA:bDriving": "BooleanAttr",
    "TAGame.Vehicle_TA:bPodiumMode": "BooleanAttr",
    "TAGame.Vehicle_TA:bReplicatedHandbrake": "BooleanAttr",
    "TAGame.Vehicle_TA:ReplicatedSteer": "ByteAttr",
    "TAGame.Vehicle_TA:ReplicatedThrottle": "ByteAttr",
    "TAGame.VehiclePickup_TA:bNoPickup": "BooleanAttr",
    "TAGame.VehiclePickup_TA:ReplicatedPickupData": "PickupAttr",
    "TAGame.VehiclePickup_TA:NewReplicatedPickupData": "PickupNewAttr",
    "TAGame.Ball_Haunted_TA:LastTeamTouch": "ByteAttr",
    "TAGame.Ball_Haunted_TA:TotalActiveBeams": "ByteAttr",
    "TAGame.Ball_Haunted_TA:DeactivatedGoalIndex": "ByteAttr",
    "TAGame.Ball_Haunted_TA:ReplicatedBeamBrokenValue": "ByteAttr",
    "TAGame.Ball_Haunted_TA:bIsBallBeamed": "BooleanAttr",
    "TAGame.SpecialPickup_Rugby_TA:bBallWelded": "BooleanAttr",
    "TAGame.Cannon_TA:Pitch": "FloatAttr",
    "TAGame.Cannon_TA:FireCount": "ByteAttr",
}
```

## Class:ParentClass

```json
{
    "Engine.Actor": "Core.Object",
    "Engine.GameReplicationInfo": "Engine.ReplicationInfo",
    "Engine.Info": "Engine.Actor",
    "Engine.Pawn": "Engine.Actor",
    "Engine.PlayerReplicationInfo": "Engine.ReplicationInfo",
    "Engine.ReplicationInfo": "Engine.Info",
    "Engine.TeamInfo": "Engine.ReplicationInfo",
    "ProjectX.GRI_X": "Engine.GameReplicationInfo",
    "ProjectX.Pawn_X": "Engine.Pawn",
    "ProjectX.PRI_X": "Engine.PlayerReplicationInfo",
    "TAGame.Ball_TA": "TAGame.RBActor_TA",
    "TAGame.CameraSettingsActor_TA": "Engine.ReplicationInfo",
    "TAGame.Car_Season_TA": "TAGame.PRI_TA",
    "TAGame.Car_TA": "TAGame.Vehicle_TA",
    "TAGame.CarComponent_Boost_TA": "TAGame.CarComponent_TA",
    "TAGame.CarComponent_Dodge_TA": "TAGame.CarComponent_TA",
    "TAGame.CarComponent_FlipCar_TA": "TAGame.CarComponent_TA",
    "TAGame.CarComponent_Jump_TA": "TAGame.CarComponent_TA",
    "TAGame.CarComponent_TA": "Engine.ReplicationInfo",
    "TAGame.CrowdActor_TA": "Engine.ReplicationInfo",
    "TAGame.CrowdManager_TA": "Engine.ReplicationInfo",
    "TAGame.GameEvent_Season_TA": "TAGame.GameEvent_Soccar_TA",
    "TAGame.GameEvent_Soccar_TA": "TAGame.GameEvent_Team_TA",
    "TAGame.GameEvent_SoccarPrivate_TA": "TAGame.GameEvent_Soccar_TA",
    "TAGame.GameEvent_SoccarSplitscreen_TA": "TAGame.GameEvent_SoccarPrivate_TA",
    "TAGame.GameEvent_Tutorial_TA": "TAGame.GameEvent_Soccar_TA",
    "TAGame.GameEvent_TA": "Engine.ReplicationInfo",
    "TAGame.GameEvent_Team_TA": "TAGame.GameEvent_TA",
    "TAGame.GRI_TA": "ProjectX.GRI_X",
    "TAGame.InMapScoreboard_TA": "Engine.Actor",
    "TAGame.PRI_Breakout_TA": "TAGame.PRI_TA",
    "TAGame.PRI_TA": "ProjectX.PRI_X",
    "TAGame.RBActor_TA": "ProjectX.Pawn_X",
    "TAGame.SpecialPickup_BallCarSpring_TA": "TAGame.SpecialPickup_Spring_TA",
    "TAGame.SpecialPickup_BallFreeze_TA": "TAGame.SpecialPickup_Targeted_TA",
    "TAGame.SpecialPickup_BallGravity_TA": "TAGame.SpecialPickup_TA",
    "TAGame.SpecialPickup_BallLasso_TA": "TAGame.SpecialPickup_Targeted_TA",
    "TAGame.SpecialPickup_BallVelcro_TA": "TAGame.SpecialPickup_TA",
    "TAGame.SpecialPickup_Batarang_TA": "TAGame.SpecialPickup_BallLasso_TA",
    "TAGame.SpecialPickup_BoostOverride_TA": "TAGame.SpecialPickup_Targeted_TA",
    "TAGame.SpecialPickup_GrapplingHook_TA": "TAGame.SpecialPickup_Targeted_TA",
    "TAGame.SpecialPickup_HitForce_TA": "TAGame.SpecialPickup_TA",
    "TAGame.SpecialPickup_Spring_TA": "TAGame.SpecialPickup_Targeted_TA",
    "TAGame.SpecialPickup_Swapper_TA": "TAGame.SpecialPickup_Targeted_TA",
    "TAGame.SpecialPickup_TA": "TAGame.CarComponent_TA",
    "TAGame.SpecialPickup_Targeted_TA": "TAGame.SpecialPickup_TA",
    "TAGame.SpecialPickup_Tornado_TA": "TAGame.SpecialPickup_TA",
    "TAGame.SpecialPickup_Rugby_TA": "TAGame.SpecialPickup_TA",
    "TAGame.Team_Soccar_TA": "TAGame.Team_TA",
    "TAGame.Team_TA": "Engine.TeamInfo",
    "TAGame.Vehicle_TA": "TAGame.RBActor_TA",
    "TAGame.VehiclePickup_TA": "Engine.ReplicationInfo",
    "TAGame.VehiclePickup_Boost_TA": "TAGame.VehiclePickup_TA",
    "TAGame.SpecialPickup_HauntedBallBeam_TA": "TAGame.SpecialPickup_TA",
    "TAGame.SpecialPickup_Football_TA": "TAGame.SpecialPickup_TA",
    "TAGame.Cannon_TA": "Engine.Actor",
    "TAGame.Ball_God_TA": "TAGame.Ball_TA",
    "TAGame.GameEvent_GodBall_TA": "TAGame.GameEvent_Soccar_TA",
}
```

## Object:Parent

```json
{
    "Archetypes.Car.Car_Default": "TAGame.Car_TA",
    "Mutators.Mutators.Mutators.FreePlay:CarArchetype": "TAGame.Car_TA",
    "Archetypes.GameEvent.GameEvent_Season:CarArchetype": "TAGame.Car_TA",
    "Archetypes.Car.Car_PostGameLobby": "TAGame.Car_TA",
    "Archetypes.Ball.Ball_Default": "TAGame.Ball_TA",
    "Archetypes.Ball.Ball_Basketball": "TAGame.Ball_TA",
    "Archetypes.Ball.Ball_BasketBall": "TAGame.Ball_TA",
    "Archetypes.Ball.Ball_BasketBall_Mutator": "TAGame.Ball_TA",
    "Archetypes.Ball.Ball_Puck": "TAGame.Ball_TA",
    "Archetypes.Ball.CubeBall": "TAGame.Ball_TA",
    "Archetypes.Ball.Ball_Beachball": "TAGame.Ball_TA",
    "Archetypes.Ball.Ball_Anniversary": "TAGame.Ball_TA",
    "Archetypes.Ball.Ball_Football": "TAGame.Ball_TA",
    "Archetypes.Ball.Ball_Ekin": "TAGame.Ball_TA",
    "Archetypes.Ball.Ball_Breakout": "TAGame.Ball_Breakout_TA",
    "Archetypes.Ball.Ball_Haunted": "TAGame.Ball_Haunted_TA",
    "Archetypes.Ball.Ball_God": "TAGame.Ball_God_TA",
    "Archetypes.CarComponents.CarComponent_Boost": "TAGame.CarComponent_Boost_TA",
    "Archetypes.CarComponents.CarComponent_Dodge": "TAGame.CarComponent_Dodge_TA",
    "Archetypes.CarComponents.CarComponent_DoubleJump": "TAGame.CarComponent_Dodge_TA",
    "Archetypes.CarComponents.CarComponent_FlipCar": "TAGame.CarComponent_FlipCar_TA",
    "Archetypes.CarComponents.CarComponent_Jump": "TAGame.CarComponent_Jump_TA",
    "Archetypes.Mutators.Mutator_Robin:DoubleJump": "TAGame.CarComponent_DoubleJump_Robin_TA",
    "Archetypes.Mutators.Mutator_Robin:Jump": "TAGame.CarComponent_Jump_Robin_TA",
    "Archetypes.Mutators.Mutator_Robin:AutoFlip": "TAGame.CarComponent_FlipCar_TA",
    "Archetypes.Teams.Team0": "TAGame.Team_Soccar_TA",
    "Archetypes.Teams.Team1": "TAGame.Team_Soccar_TA",
    "Archetypes.Teams.TeamWhite0": "TAGame.Team_Soccar_TA",
    "Archetypes.Teams.TeamWhite1": "TAGame.Team_Soccar_TA",
    "TAGame.Default__PRI_TA": "TAGame.PRI_TA",
    "Archetypes.GameEvent.GameEvent_Basketball": "TAGame.GameEvent_Soccar_TA",
    "Archetypes.GameEvent.GameEvent_Hockey": "TAGame.GameEvent_Soccar_TA",
    "Archetypes.GameEvent.GameEvent_Soccar": "TAGame.GameEvent_Soccar_TA",
    "Archetypes.GameEvent.GameEvent_Items": "TAGame.GameEvent_Soccar_TA",
    "Archetypes.GameEvent.GameEvent_SoccarLan": "TAGame.GameEvent_Soccar_TA",
    "Archetypes.GameEvent.GameEvent_SoccarPrivate": "TAGame.GameEvent_SoccarPrivate_TA",
    "Archetypes.GameEvent.GameEvent_BasketballPrivate": "TAGame.GameEvent_SoccarPrivate_TA",
    "Archetypes.GameEvent.GameEvent_HockeyPrivate": "TAGame.GameEvent_SoccarPrivate_TA",
    "Archetypes.GameEvent.GameEvent_SoccarSplitscreen": "TAGame.GameEvent_SoccarSplitscreen_TA",
    "Archetypes.GameEvent.GameEvent_BasketballSplitscreen": "TAGame.GameEvent_SoccarSplitscreen_TA",
    "Archetypes.GameEvent.GameEvent_HockeySplitscreen": "TAGame.GameEvent_SoccarSplitscreen_TA",
    "Archetypes.GameEvent.GameEvent_Season": "TAGame.GameEvent_Season_TA",
    "Archetypes.GameEvent.GameEvent_Breakout": "TAGame.GameEvent_Breakout_TA",
    "ProjectX.Default__NetModeReplicator_X": "ProjectX.NetModeReplicator_X",
    "TAGame.Default__CameraSettingsActor_TA": "TAGame.CameraSettingsActor_TA",
    "Archetypes.SpecialPickups.SpecialPickup_GravityWell": "TAGame.SpecialPickup_BallGravity_TA",
    "Archetypes.SpecialPickups.SpecialPickup_BallVelcro": "TAGame.SpecialPickup_BallVelcro_TA",
    "Archetypes.SpecialPickups.SpecialPickup_BallLasso": "TAGame.SpecialPickup_BallLasso_TA",
    "Archetypes.SpecialPickups.SpecialPickup_BallGrapplingHook": "TAGame.SpecialPickup_GrapplingHook_TA",
    "Archetypes.SpecialPickups.SpecialPickup_Swapper": "TAGame.SpecialPickup_Swapper_TA",
    "Archetypes.SpecialPickups.SpecialPickup_BallFreeze": "TAGame.SpecialPickup_BallFreeze_TA",
    "Archetypes.SpecialPickups.SpecialPickup_BoostOverride": "TAGame.SpecialPickup_BoostOverride_TA",
    "Archetypes.SpecialPickups.SpecialPickup_Tornado": "TAGame.SpecialPickup_Tornado_TA",
    "Archetypes.SpecialPickups.SpecialPickup_CarSpring": "TAGame.SpecialPickup_BallCarSpring_TA",
    "Archetypes.SpecialPickups.SpecialPickup_BallSpring": "TAGame.SpecialPickup_BallCarSpring_TA",
    "Archetypes.SpecialPickups.SpecialPickup_StrongHit": "TAGame.SpecialPickup_HitForce_TA",
    "Archetypes.SpecialPickups.SpecialPickup_Batarang": "TAGame.SpecialPickup_Batarang_TA",
    "Archetypes.SpecialPickups.SpecialPickup_HauntedBallBeam": "TAGame.SpecialPickup_HauntedBallBeam_TA",
    "Archetypes.SpecialPickups.SpecialPickup_Rugby": "TAGame.SpecialPickup_Rugby_TA",
    "gameinfo_godball.GameInfo.gameinfo_godball:Archetype": "TAGame.GameEvent_GodBall_TA",
    "GameInfo_GodBall.GameInfo.GameInfo_GodBall:Archetype": "TAGame.GameEvent_GodBall_TA",
    "TAGame.Default__MaxTimeWarningData_TA": "TAGame.MaxTimeWarningData_TA",
    "TAGame.Default__RumblePickups_TA": "TAGame.RumblePickups_TA",
    "Archetypes.SpecialPickups.SpecialPickup_Football": "TAGame.SpecialPickup_Football_TA",
    "GameInfo_FootBall.GameInfo.GameInfo_FootBall:Archetype": "TAGame.GameEvent_Football_TA",
    "TAGame.Default__PickupTimer_TA": "TAGame.PickupTimer_TA",
    "TAGame.Default__PRI_Breakout_TA": "TAGame.PRI_Breakout_TA",
    "Archetypes.GameEvent.GameEvent_FTE_Part1_Prime": "TAGame.GameEvent_FTE_TA",
    "GameInfo_Tutorial.GameEvent.GameEvent_Tutorial_Aerial": "TAGame.GameEvent_Training_Aerial_TA",
    "Archetypes.Ball.Ball_Training": "TAGame.Ball_Tutorial_TA",
    "Archetypes.Tutorial.Cannon": "TAGame.Cannon_TA",
}
```

# Unknowns

Below is a list of all currently unknown portions of the replay file.

- `Unknown 01` :: `64` bits that appear between a header property's property type and value.
- `Unknown 02` :: `1` bit that appears at the end of the `DamageState` attribute.
- `Unknown 03` :: `1` bit that appears between the deserialized `Explosion` and secondary actor ID in an `ExtendedExplosion` attribute.
- `Unknown 04` :: `32` bits that appear after the **topper** of a `Loadout` attribute.
- `Unknown 05` :: `32` bits that appear after `unknown_04`, only if the `Loadout`'s **version** is >= `9`.
- `Unknown 06-08` :: `3` x `32` bits that appear at the end of the `Loadout` attribute if the `Loadout`'s **version** is >= `16`.
- `Unknown 09` :: 1 bit that appears at the beginning of a `Product`.
- `Unknown 10-11` :: `2` x `1` bit that appears at the end of the `LoadoutsOnline` attribute.
- `Unknown 12-13` :: `2` x `1` bit that appears at the end of the `PickupInfo` attribute.
- `Unknown 14` :: `1` bit that appears at the beginning of a `RepStatTitle` attribute.
- `Unknown 15` :: `1` bit that appears after the **name** of a `RepStatTitle` attribute.
- `Unknown 16-17` :: `2` x `1` bit that appears after the **name** of a `Reservation` attribute.
- `Unknown 18` :: `6` bits that appear at the end of a `Reservation` attribute so long as `ENGINE_VERSION` >= `868` AND `LICENSEE_VERSION` >= `12`.
- `Unknown 19` :: `8` bits (or `16` if `NET_VERSION` >= `1`) that appear between the **name** and **online_id** of a `PlayStation` remote ID.
- `Unknown 20` :: `24` bytes that appear after the **online_id** of a `PsyNet` remote ID, only if `NET_VERSION` >= `10`.
- `Unknown 21` :: `24` bytes that appear after the **online_id** of a `Switch` remote ID.
- `Unknown 22` :: `1` bit that appears at the beginning of a `StatEvent` attribute.
- `Unknown 23-30` :: `2` x `1` bit, then `5` x `32` bits, then `1` bit that make up the `Title` attribute.
- `Unknown 31` :: `1` bit that appears after the **name_id** of a new actor.

# Acknowledgements

I would like to thank [Nick Babcock](https://github.com/nickbabcock) for their direct help during my journey of learning the Rocket League replay format. [Check out their Rocket League replay parser here](https://github.com/nickbabcock/boxcars).

The [`Object:SpawnTrajectory`](#objectspawntrajectory), [`Object:AttributeType`](#objectattributetype), and [`Class:ParentClass`](#classparentclass) hash maps came from [boxcars](https://github.com/nickbabcock/boxcars), specifically the [data file](https://github.com/nickbabcock/boxcars/blob/master/src/data.rs). The [`Class:ParentClass`](#classparentclass) hash map is slightly modified.

The [`Object:Parent`](#objectparent) hash map is a slightly modified version of [this portion of the `ActorState` file](https://github.com/jjbott/RocketLeagueReplayParser/blob/9daa4e0491b3a48c9267eeeb93d065ae1b0f35a9/RocketLeagueReplayParser/NetworkStream/ActorState.cs#L46-L183) from [RocketLeagueReplayParser](https://github.com/jjbott/RocketLeagueReplayParser).
