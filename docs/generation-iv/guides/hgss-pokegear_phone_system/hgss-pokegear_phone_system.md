---
title: Pokegear Phone System
tags:
  - Guide (HeartGold)
  - Guide (SoulSilver)
---

# Pokegear Phone System

> Author(s): [MrHam88](https://github.com/DevHam88/).  
> Research: [pret/pokeheartgold](https://github.com/pret/pokeheartgold), Eclipse (Luna), Senate, BlackShark.  

This guide explains the Pokegear phone system in Pokemon HeartGold and SoulSilver: registering contacts, ordinary outgoing calls, random incoming calls, story/event calls, text archives, trainer rematches, Gym Leader rematches, and post-call gifts.  

The system is spread across several files. The most important distinction is that a **contact**, a **phone conversation**, and a **field script** are different things. A contact is a row in `pmtel_book.dat`; it can point to a message archive and a call handler, but story calls and item collection often use normal field scripts as well.

---

## Overview

HGSS's Pokegear phone system is a set of connected but separate systems. `pmtel_book.dat` is a fixed contact list that defines each contact''s ID, handler type, trainer battle ID, home map, schedule, sorting data, and call settings; save data records which contacts the player has registered plus pending rematches, gifts, and queued calls. When the player calls a number, or when the game decides to ring, the Pokegear code chooses a contact-specific handler: ordinary trainers use compact Overlay 101 header and definition tables to select a message and possibly queue a rematch or gift, while Mom, Elm, Oak, Gym Leaders, Baoba, and other special contacts use dedicated logic. The text itself comes from that contact''s message archive, whose message `0` supplies the contact name. Incoming calls share a timed overworld check: it handles queued system events first, then may choose an eligible registered caller at random; map-header permissions can block calls. Ordinary trainer rematches use the contact''s trainer ID to look up a fixed six-entry battle-ID row in Overlay 26, while field scripts can register contacts or immediately configure and start story calls such as Elm''s stolen-Pokemon call.

The following tables summarise the main data and call paths:

| Type | System | Location | What it stores |
|---|---|---|---|
| Data | Pokegear contact data | `tel/pmtel_book.dat` | Contact type, base trainer ID, map location, item, local-call script, schedule, and random-call bucket. |
| Data | Phone conversation texts | `sPhoneMessageGmm` | The message archive used for that contact's name and phone text. |
| Data | Rematch trainer ID lookup | Overlay 26, file offset `0x20C` | The ordinary trainer rematch battle IDs. |
| Game code | Phone call code | ARM9 and Overlays 2, 26, and 101 | The logic that chooses dialogue for the current contact, direction, map, day, time, and story state. |
| Save file | Saved Pokegear phonebook | N/A | Whether the player has registered the contact. |
| Save file | Persistent phone state | N/A | Pending rematches, pending gifts, and queued event calls. |


The game has three call paths:

| Call path | Direction | Example |
|---|---|---|
| Player selects a number | Outgoing | Calling Elm from the Pokegear menu option. |
| The ring manager chooses a registered contact | Incoming | Youngster Joey calling with ordinary chatter, a rematch, or an item. |
| A field script or system event starts/queues a call | Incoming/event | Elm's stolen-Pokemon call, the Togepi Egg call, Mom's purchase call, Bill's full-PC call. |

---

## Phonebook data

The ROM file is a direct NitroFS file: `tel/pmtel_book.dat`

It is **not** a NARC member. For binary hacking, extract and edit this file directly.

### File layout

The first four bytes are a little-endian entry count (75). They are followed by 20-byte contact records for each entry.

| Offset | Size (bytes) | Decomp field | Meaning |
|:---:|:---:|---|---|
| `00` | 1 | `id` | Contact ID (`PHONE_CONTACT_*`). |
| `01` | 1 | `type` | Phone call handler type. Values 0-15 select: generic, Mom, Elm, Gym Leader, and other specialised call logic; see [Contact type values](#contact-type-values). |
| `02` | 1 | `unk2` | Unknown. No direct reader was found in the current public decomp, but that is not proof it is unused in vanilla binary code; leave it unchanged. |
| `03` | 1 | `trainerClass` | Title label shown in the Pokegear contact list. Standard trainer classes and special phone-contact labels use different text paths; see [Contact-list class text](#contact-list-class-text). |
| `04` | 2 | `trainerId` | The contact's original trainer battle ID, or `TRAINER_NONE` (`0`) for a non-trainer. Ordinary trainer rematches use this to find an overlay 26 rematch lookup row. |
| `06` | 2 | `mapId` | The contact's **map-header ID**. It controls local-call behaviour, random-call filtering, location text, and the limited rematch/gift map marker; see [What `mapId` controls](#what-mapid-controls). |
| `08` | 2 | `gift` | Static item used by the trainer-gift field-script path. |
| `0A` | 2 | `phoneScriptIfLocal` | Generic contact's outgoing phone script when the player is on this contact's `mapId`. |
| `0C` | 1 | `unkC` | Greeting-set index. Values `0`-`7` select a row from the shared phone-greeting table; `FF` suppresses the greeting. |
| `0D` | 1 | `rematchWeekday` | Weekday used by the ordinary phone-trainer schedule and Gym Leader schedule. It is also gated by story/state checks; see [Schedule and story gates](#schedule-and-story-gates). |
| `0E` | 1 | `rematchTimeOfDay` | Morning `0`, day `1`, or night `2`, used with the weekday by those schedule checks. |
| `0F` | 1 | `unkF` | Random incoming-call "probability bucket". The game chooses bucket `0` (50% of the time), bucket `1` (30% of the time), and bucket `2` (20% of the time). |
| `10` | 1 | `sortParam[0]` | Stored Pokegear title-sort rank. |
| `11` | 1 | `sortParam[1]` | Stored Pokegear alphabetical-sort rank. |
| `12` | 1 | `sortParam[2]` | Stored Pokegear location-sort rank. |
| `13` | 1 | `sortParam[3]` | Padding (`00` in vanilla). |

:::caution  
The first four bytes control how many 20-byte records the loader reads and allocates, however, they are **not** a safe "add a new contact" control by themselves.

The game has a hardcoded `NUM_PHONE_CONTACTS = 75` limit. Contact IDs index fixed-size save arrays for registered numbers and `PhoneRematch` state, the phone UI slot data, the contact-to-message-archive table, generic-header blocks, and other tables. Several call paths reject IDs `>= 75` before indexing the phonebook.

Appending a record and increasing the count can make the file loader allocate and read it: it does not make the record usable as a new phone contact. A real contact expansion requires coordinated binary patches and expansions of every contact-ID-indexed table.  
:::

### Contact type values

`type` is a numeric call-handler selector, not a trainer class and not a random-call frequency. The vanilla values are:

| Value | Decomp name | Contacts / effect |
|:---:|---|---|
| `0` | `GENERIC` | Ordinary phone trainers. Uses generic header/script tables. |
| `1` | `MOM` | Mom's savings and gift-queue handler. |
| `2` | `PROF_ELM` | Elm's story-flag, badge, Egg, and Pokérus handler. |
| `3` | `PROF_OAK` | Oak's dedicated handler. |
| `4` | `KURT` | Kurt's dedicated handler. |
| `5` | `BIKE_SHOP` | Bike Shop handler. |
| `6` | `KENJI` | Black Belt Kenji handler. |
| `7` | `BILL` | Bill handler. |
| `8` | `DAYCAREMAN` | Day-Care Man handler. |
| `9` | `DAYCARELADY` | Day-Care Lady handler. |
| `10` | `BUENA` | Buena handler. |
| `11` | `ETHAN_LYRA` | Childhood-friend handler. |
| `12` | `GYMLEADER` | Fighting Dojo/Gym Leader rematch handler. |
| `13` | `BAOBA` | Safari Zone Warden handler. |
| `14` | `IRWIN` | Juggler Irwin handler. |
| `15` | `UNK15` | Unnamed/unknown handler. |

Changing this byte changes which ARM handler reads the contact. For example, changing an ordinary trainer from 0 to 12 does not create a Gym Leader rematch: the Gym Leader handler expects Gym Leader message layouts and Fighting Dojo state.

### Title

The title label in the phone list is selected from the `trainerClass` text archive; or text archive `271`; or is empty.

| title (`trainerClass` in decomp) range/value | Source of the displayed label |
|---|---|
| `0`-`199` | Normal trainer-class formatter. Vanilla class-name data is populated only through class `128`; a hack using `129`-`199` must also supply normal class-name text. |
| `200` (`PHONE_MOM`) | Empty label. Mom displays no trainer class. |
| `201`-`207` | Message archive 271, starting at message 38 plus `trainerClass - 201`. |

The special values are:  
- 201 Pokemon Professor
- 202 Childhood Friend
- 203 Poke Ball Creator
- 204 Day-Care
- 205 Radio Personality
- 206 Poke Maniac
- 207 Safari Warden

Elm and Oak both display "Pokemon Professor", while Ethan/Lyra display "Childhood Friend" for example. The name itself still comes from message `0` of that contact's own phone text archive.

#### Compatibility with expanded trainer classes

<details>
<summary>Click for details...</summary>

This is a real compatibility limit for hacks that add trainer classes. The phone UI does not ask the normal trainer-class-name system whether a class is valid. Its logic is hardcoded as:

```text
if trainerClass == 200:
    show no class label                 # Mom
else if trainerClass >= 201:
    show message 38 + (trainerClass - 201) from archive 271
else:
    show the normal trainer-class name
```

Therefore the normal-class branch of an unpatched Pokegear phone UI accepts **0-199**, although vanilla class-name data exists only for `0`-`128`. Values 200-207 are reserved for non-trainer phone contacts, but the comparison actually catches **every value from 201 upward**.

:::caution  
If a hack adds normal trainer classes at 200 or above, changing the trainer-class text archive alone is insufficient.  
:::

A hypothetical trainer class **210** would be interpreted as a special phone title and reads message `38 + (210 - 201) = 47` from archive 271. It will display whatever message 47 contains; it does not gain a normal class name merely because a hack added class 210 elsewhere.

Separately, `trainerClass` is one unsigned byte. A phonebook row can therefore store only `0` through `255` as either a normal trainer-class ID or a special phone-contact title selector. Expanding normal trainer classes and moving the special values above them cannot extend this field beyond `255` without changing the phonebook record format and every reader of it.

</details>

### Trainer IDs for trainers and non-trainers

Vanilla non-trainer contacts use `TRAINER_NONE` (`0`), in the `trainerId` field: Mom, Elm, Oak, Ethan/Lyra, Kurt, the Day-Care couple, Buena, Bill, and the other special contacts do not carry a "real" trainer battle ID.

Trainer contacts carry their original battle ID, regardless of whether the contact can ever be rematched. The battle/rematch code decides whether that ID has a usable rematch row; the gift field and phone-item state do not require a rematch lookup. In the vanilla contact table there are 63 rows which do **not** have trainerId of `TRAINER_NONE`, matching the 63 overlay 26 rematch sets. Custom non-trainer contacts should use `TRAINER_NONE` (`0`) when no field trainer exists.

### What `mapId` controls

`mapId` has several proven uses:

- When a phone conversation is initialised, the location name of the header in `mapId` is stored in a message buffer (`3`), to be potentially used in conversations. The player's current location name is similarly stored in buffer `2`.
- Random incoming calls exclude a contact when the player is on the same `mapId`.
- Most special handlers and the generic handler use a same-map check to select their "local" response.
- Every type-0 (generic) phone call prepares two Pokemon-name message buffers before it prints its selected archive message: buffer `10` receives a random member of the caller's `trainerId` battle party, and buffer `11` receives a species selected from the wild-encounter data for this `mapId`. For buffer `11`, a contact whose `trainerClass` byte is exactly `11` (`0x0B`, Fisherman) selects one of the five Good Rod slots (with the night-fishing replacement at night); all other values select one of the twelve land slots for the current morning/day/night. A map with no wild encounters supplies Rattata instead. This is implemented by `GearPhoneCall_Generic` and `getRandomEncounterSlot` in Pokegear Overlay 101's generic phone-script code.
- If the contact's map-header ID is exactly `96` (`0x0060`, `MAP_NATIONAL_PARK`), the random incoming-call manager excludes that contact when flag `0x996` (`2454`, decomp name: `FLAG_UNK_996`) is set. The decomp calls its direct helper `Save_VarsFlags_CheckBugContestFlag`. The generic rematch/gift header conditions use the same check.
- The Pokegear map shows its pending battle marker when a **generic** (`type = 0`) contact at that `mapId` has either a valid ordinary rematch or a queued **dynamic** phone gift.

:::info  
The Pokegear phone book's location sort uses the stored location rank at `sortParam[2]`; it does not derive an order from `mapId` at runtime.
:::

---

## Registering phone numbers

The static phonebook table does not immediately and fully populate contacts in the player's Pokegear. Registration is saved separately in `SavePokegear.phoneContacts[]`. Empty slots are `FF`; registered slots contain contact IDs.

Normal field scripts use the following command IDs. The first name is the established scrcmd/DSPRE name; the second is the decomp name for the same command.

| Opcode | scrcmd / editor name | Decomp name | Use |
|---|---|---|---|
| `0x0092` | `RecordPokegearNumber contact` | `RegisterGearNumber contact` | Add a valid contact ID to the player's phonebook. |
| `0x0093` | `CheckPokegearNumberRegistered contact result` | `CheckRegisteredPhoneNumber contact result` | Write 0 or 1 depending on registration state. |

### Sorting phone numbers

There are a number of ways of sorting phone numbers in the Pokegear. The first three use a stored one-byte rank in each contact record, rather than recalculating from the displayed text or `mapId`:
- Title (`sortParam[0]`)
- Alphabet (`sortParam[1]`)
- Location (`sortParam[2]`)
- Manual arrangement

---

## Incoming and outgoing calls

Incoming and outgoing calls use different map permissions and different dialogue selection paths.

| Direction | Map-header field | Engine check | What it controls |
|---|---|---|---|
| Outgoing | `outgoingCalls` | `MapHeader_CanPlacePhoneCalls` | Whether the player can successfully start a call from the Pokegear. |
| Incoming | `incomingCalls` | `MapHeader_CanReceivePhoneCalls` | Whether random callers and queued event calls can ring on the current map. |

These are independent one-bit fields in `MapHeader`. A map can permit outgoing calls while blocking incoming calls.

DSPRE's map-header editor can set or clear `outgoingCalls` and `incomingCalls` independently.

### Outgoing

An outgoing call begins when the player selects a registered contact in the Pokegear. `outgoingCalls` must be enabled in the current map header. The selected contact's `type`, `mapId`, and, for type-0 contacts, the generic outgoing phone-script headers determine the conversation; this is distinct from the incoming random-call and queued-event paths below.

### Incoming: time trigger

The game keeps a runtime counter of elapsed **real-time minutes** for incoming calls. This timing gate is shared by queued system-event calls and ordinary random calls. While the player is in the normal overworld and no field task is running, completed minutes are added to this counter. At `10` minutes, the next normal input update when the player is standing still or has finished a movement checks for a call. It is not based on steps, frames, or a random per-step chance.

Queued system-event calls are checked first. Their triggering code can set the counter to one minute below the current threshold, making the queued call eligible after the next completed minute; this uses the threshold dynamically. A map which disallows incoming calls does not clear the counter; the check instead waits until the player reaches an allowed map. The timer is runtime state, so it is reset when a call is accepted or otherwise cleared.

#### Changing the shared incoming-call threshold (US HeartGold)

The vanilla ten-minute interval can be changed with two ARM9 immediate-byte edits. Both values must be changed to the same whole-minute value `N` (`1`-`255`). These are US HeartGold ARM9 **file offsets**; verify the original bytes before using them on another revision.

| ARM9 file offset | RAM address | Vanilla bytes | Meaning | Change for `N` minutes |
|:---:|:---:|---|---|---|
| `0x92DB2` | `0x02092DB2` | `0A 20` | Sets the call interval to `10` minutes. | Replace `0A` with `N`; retain `20`. |
| `0x92E44` | `0x02092E44` | `0A 29` | Compares the first elapsed-time update against `10` minutes. | Replace `0A` with `N`; retain `29`. |

For example, to use twenty minutes, change `0A 20` to `14 20` at `0x92DB2` and `0A 29` to `14 29` at `0x92E44`. The second edit keeps the first-update safeguard consistent with the interval. A queued trigger which sets the timer to one minute below the threshold still becomes eligible after one completed minute, because it uses the changed threshold value dynamically; a queued trigger which does not expedite itself waits for the shared threshold.

The shared incoming-call check:

1. Rejects maps where incoming calls are disabled.
2. Handles queued event calls first.
3. Continues to the ordinary random-call branch only when no queued call is started.

---

#### Incoming: random frequency

The following behavior applies only to the ordinary random-call branch:

1. Makes a `51%` roll (`0`-`50` inclusive out of `0`-`99`). A failed roll clears the elapsed-minute counter, so the next ordinary attempt is after another ten minutes.
2. Chooses an `unkF` bucket: 0 (50%), 1 (30%), or 2 (20%).
3. Keeps only registered contacts whose `type` is `0` (generic trainer), `10` (Buena), `11` (Ethan/Lyra), `12` (Gym Leader), or `14` (Irwin). All other types are excluded from this ordinary random-caller pool, though they may still be called by the player, start an event call, or be started by a field script.
4. Removes contacts on the player's current map, contacts already recorded in that bucket's call history, Buena in unsuitable conditions, and National Park contacts during the Bug-Catching Contest.
5. Randomly chooses from the remaining contacts. If no candidate remains, it clears the elapsed-minute counter.

The call starts with `isScriptedCall = 0`. A type-0 contact uses the generic dialogue logic; types 10, 11, 12, and 14 each use their own dedicated handler.

##### Where the allowed-type list is defined

<details>
<summary>Click for details...</summary>

At US decompressed Overlay 2 file offset `0xC45C` (RAM `0x02251FDC`), the random-candidate builder contains this literal chain of five comparisons:

```text
type == 0 || type == 12 || type == 11 || type == 10 || type == 14
```

For the US decompressed Overlay 2, this function starts at **file offset `0xC45C`** (RAM `0x02251FDC`), using Overlay 2's RAM base `0x02245B80`. The comparisons are executable Thumb code, not five adjacent data bytes. Adding another allowed `type` therefore requires a binary code patch to this function, not an edit to the contact record alone.

</details>

##### What "recently called" means

<details>
<summary>Click for details...</summary>

The random-caller history is persistent save data, not a timer. It is the eight-byte `SAVE_MISC_DATA.unk_0280` field at structure offset `0x280`, with `FF` marking an unused slot. The game partitions it by `unkF` bucket:

| Chosen bucket | History bytes in `SAVE_MISC_DATA + 0x280` | Number of remembered contact IDs |
|:---:|---|:---:|
| `0` | `00`-`03` | `4` |
| `1` | `04`-`05` | `2` |
| `2` | `06`-`07` | `2` |

The candidate builder checks only the history range for the bucket chosen at step 3. A matching contact ID is excluded. If the history already contains every current eligible candidate in that bucket, it clears that bucket's history before filtering so that the pool cannot become empty forever. When the player accepts an ordinary random incoming call, the selected contact ID is appended to that bucket's range; if it is full, the oldest entry is discarded. Queued event and field-scripted calls do not use this history.

The filter is at US Overlay 2 file offset `0xC45C` / RAM `0x02251FDC`; the save helpers are identified by the decomp as `sub_0202AA44`, `sub_0202AA9C`, `sub_0202AAD4`, and `sub_0202AB18` in `src/save_misc.c`. The actual save-file location of the containing `SAVE_MISC_DATA` block is outside the scope of this page; use its structure-relative offset rather than assuming a fixed offset in every save file.

</details>

---

#### Incoming: queued system-event trigger

Queued event calls are not ordinary random calls and are not entries in the normal event-flag or variable systems. `PhoneCallPersistentState.callTriggerFlags[2]` is a separate two-byte packed bitfield in the phone-call persistent save data. Bits `0` through `12` are the 13 trigger IDs below.  

In vanilla, native game code sets these bits when it detects its own condition, such as an Egg hatching, cycling 1,024 steps, or a full PC. Field scripts can also set one with unnamed command `ScrCmd_148` (`0x0094`): it takes two bytes, `triggerId` (`0`-`12`) and `expedite` (`0` or nonzero). It sets that trigger bit; when `expedite` is nonzero, it advances the shared incoming-call timer to one minute before its threshold if necessary. `UnsetPhoneCallTrigger` (`0x0095`) takes one `triggerId` byte and clears that bit.  

The caller, predefined phone script, and pickup behavior do not live in the bitfield. They come from a separate 13-record table in Overlay 2 (`ov02_02253C84` in `src/field/overlay_2_gear_phone.c`). Its RAM address `0x02253C84` converts to **file offset `0xE104`**. It occupies `0x4E` bytes: `0xE104` through `0xE151` inclusive. Record `n` is the definition for trigger bit `n`; its file offset is `0xE104 + (n * 6)`.

```text
callerId (u8), unknown (u8), phoneScriptId (u16 little-endian), forcePickUp (u8), unknown (u8)
```

| Bit | File offset | Trigger ID | Caller ID | Phone script ID | Condition that queues it | `forcePickUp` |
|:---:|:---:|---|:---:|:---:|---|:---:|
| `0` | `E104` | `ELM_EGG_HATCHED` | `1` Elm | `000D` | Gift Togepi Egg hatches. | `0` |
| `1` | `E10A` | `ELM_POKERUS` | `1` Elm | `0007` | Pokérus is discovered. | `0` |
| `2` | `E110` | `BIKE_SHOP_STEPS` | `15` Bike Shop | `0055` | 1,024 bicycling steps, before the Bike Shop call flag is set. | `1` |
| `3` | `E116` | `BILL_PC_FULL` | `9` Bill | `005D` | Every PC storage slot is full; the one-time Bill condition allows it. | `1` |
| `4` | `E11C` | `OAK_DEX_PROGRESS` | `2` Oak | `0000` | A National Dex ownership milestone is reached before its acknowledgement flag is set. | `0` |
| `5` | `E122` | `DAYCARE_HAS_EGG` | `6` Day-Care Man | `0000` | An egg is available and the Day-Care call flags allow it. | `0` |
| `6` | `E128` | `BAOBA_NEW_POKEMON` | `24` Baoba | `0000` | A new Safari Zone species/area arrangement is available. | `0` |
| `7` | `E12E` | `BAOBA_NEXT_TEST` | `24` Baoba | `008E` | Safari Zone test progression requests the next test. | `1` |
| `8` | `E134` | `BAOBA_OBJECT_ARRANGEMENT` | `24` Baoba | `008F` | Safari Zone object-arrangement progression. | `1` |
| `9` | `E13A` | `BAOBA_MORE_OBJECTS` | `24` Baoba | `0090` | Safari Zone more-objects progression. | `1` |
| `10` | `E140` | `BAOBA_EVEN_MORE_OBJECTS` | `24` Baoba | `0091` | Safari Zone even-more-objects progression. | `1` |
| `11` | `E146` | `BAOBA_MEMORY_LOSS` | `24` Baoba | `0092` | Safari Zone memory-loss progression. | `1` |
| `12` | `E14C` | `MOM_BOUGHT_SOMETHING` | `0` Mom | `001B` | Mom spends the player's saved money on an item. | `0` |

When a set trigger is selected, the ring manager reads its matching table record, assigns the caller and `phoneScriptId`, and marks the call as an event call (`isScriptedCall = 3`). The `forcePickUp` byte decides what happens next:

| `forcePickUp` | Result |
|:---:|---|
| `0` | The phone rings normally. The player opens/answers the Pokegear phone as usual. |
| `1` | The game begins field scene script `0x7FF` immediately. The player does not get a normal opportunity to ignore the ring. |

The table above accounts for all 13 triggers. Its `forcePickUp = 1` rows are Bike Shop, Bill, and the five Baoba progression calls; the other six ring normally. In the US file, both unknown bytes are `00` in every record; their purpose is not established, so leave them unchanged.

:::warning  
These offsets and byte values are for the US vanilla *decompressed* Overlay 2. Another version or language must be mapped independently.  
:::

:::caution No in-place space for a fourteenth event-call record  
The 13-record table ends at US decompressed Overlay 2 offset `0xE151`, but the six following zero-looking bytes are **not** free space. The Celebi cutscene's live `VecFx32 { 0, 0x1000, 0 }` constant begins at `0xE154` and occupies `0xE154`-`0xE15F`; it is used by `CelebiCutscene_SwirlEffect` in `src/field/event_cutscene_celebi.c`. Its first `u32` is zero, which makes `0xE154`-`0xE157` appear unused in a hex editor. Writing record 13 at `0xE152` would overwrite that vector's `x` component.

Only `0xE152`-`0xE153` are alignment padding. A real fourteenth record requires relocating or otherwise expanding the table and repointing its Overlay 2 readers; this page does not yet provide that relocation patch.  
:::

#### Accepting trigger ID 13 (US version)

The existing two-byte `callTriggerFlags` field can store bit `13`; it does not require a save-format change. Before a relocated record 13 can work, six immediate-byte edits must change the valid trigger count from `13` (`0D`) to `14` (`0E`). These edits make `SetPhoneCallTrigger 13, ...` set the bit, make the incoming selector scan it, and make answering the call clear it.

| File | File offset | Vanilla bytes | Replacement bytes | Purpose |
|---|:---:|---|---|---|
| ARM9 | `0x2F01E` | `0D 29` | `0E 29` | Accept trigger ID `13` in the persistent-bit setter. |
| ARM9 | `0x2F052` | `0D 29` | `0E 29` | Accept trigger ID `13` in the persistent-bit clearer. |
| ARM9 | `0x2F08E` | `0D 29` | `0E 29` | Read trigger bit `13` in the persistent-bit checker. |
| Overlay 2 | `0xC6A2` | `0D 21` | `0E 21` | Allocate 14 bytes for queued trigger candidates. |
| Overlay 2 | `0xC6AC` | `0D 22` | `0E 22` | Clear all 14 candidate bytes. |
| Overlay 2 | `0xC6FA` | `0D 2C` | `0E 2C` | Scan trigger IDs `0` through `13`. |

The ARM9 comparisons are the set, clear, and check helpers in `src/save_pokegear.c`; the Overlay 2 values are in `ov02_02252218` in `src/field/overlay_2_gear_phone.c`. The script command `0x0094` already accepts its trigger ID as a byte, so no command-format change is required.

**Test after the table itself has been safely relocated:** define record 13 at its new location, use `0x0094` with `triggerId = 13` and a nonzero expedite byte, then wait on an incoming-call-enabled map. The intended caller must ring, answering must clear the bit, and the same call must not recur after save/reload. This test proves the new index's set, scan, and clear paths; it does not validate a table relocation on its own.

#### Vanilla trigger setters (decomp references)

<details>
<summary>Click for details...</summary>

The event-record table above defines what each pending bit does. This table instead records every vanilla path which **sets** a bit. It is a source map for advanced hackers: most paths call `sub_02092E14`, which sets the persistent bit and, when its final argument is true, advances the shared incoming-call timer. Bit `1` is set by a field script, and bit `6` writes the persistent bit directly.

| Bit | Vanilla setter and condition | Expedites the incoming timer? |
|:---:|---|:---:|
| `0` | `sub_02093134` in `src/unk_02092BE8.c`, called by `src/hatch_egg_task.c` after an egg hatches. It sets the bit only when `MonIsFromTogepiEgg` identifies the special Elm Egg. | Yes |
| `1` | New Bark's field script, `files/fielddata/script/scr_seq/scr_seq_0003.s`, uses `ScrCmd_148 CALL_TRIGGER_ELM_POKERUS, FALSE` after its Pokerus check. This is the vanilla example of a script setting a queued-call bit. | No |
| `2` | `FieldSystem_UpdateBikeShop` in `src/field/field_control.c`: the Bike Shop call has not already happened, bit `2` is not already pending, and `GAME_STAT_STEPS_BIKED` is at least `1024`. | Yes |
| `3` | `sub_02093070` in `src/unk_02092BE8.c`: Bill is registered, the full-PC call flag is clear, and every PC storage slot is full. This helper is called after several battle and storage-related return paths. | Yes |
| `4` | `sub_020930C4` in `src/unk_02092BE8.c`: Oak is registered, the National Dex owned count has reached a nonzero multiple of `50`, and the corresponding Oak acknowledgement flag is clear. | No |
| `5` | `sub_0209316C` in `src/unk_02092BE8.c`, called when the Day-Care generates an Egg in `src/get_egg.c`: the Day-Care Man is registered, and the script-flag guard permits the Egg call. | Yes |
| `6` | `SaveData_SafariZone_CheckAreasWithUpdatedEncounters` in `src/unk_02097268.c`: after at least one day has passed outside an active Safari session, it compares the Safari Zone's encounter lists before and after the daily object-level update. It sets bit `6` if an area now has changed encounters, and clears it when none do. | No |
| `7` | `sub_02092E54` in `src/unk_02092BE8.c`: no Baoba bits `7`-`11` are already pending, Safari progress variable `VAR_UNK_4057` is `3`, and the Safari Zone's three-hour timer has elapsed. | Yes |
| `8` | `sub_02092E54`: Safari progress is at least `6`, the player has the National Dex, the three-hour timer reports its ordinary elapsed state, and the Safari object-unlock level is `0`. | Yes |
| `9` | `sub_02092E54`: the same post-National-Dex three-hour branch, with object-unlock level `1` or `2`. | Yes |
| `10` | `sub_02092E54`: the same branch, with object-unlock level `3`; it is also chosen by the routine's exceptional elapsed-time result when the unlock level is at least `3`. | Yes |
| `11` | `sub_02092E54`: the routine's exceptional elapsed-time result when the object-unlock level is below `3`. | Yes |
| `12` | `src/battle/battle_setup.c` calls `MomGift_TryEnqueueGiftOnBalanceChange` after Mom's savings balance changes. If that helper successfully queues a purchase, it sets bit `12`. The purchase and its quantity are stored in Mom's separate gift queue. | Yes |

`VAR_UNK_4057` is a normal script variable read by the Baoba evaluator; it is distinct from the queued-call bitfield. The three-hour test is `sub_0202F798` in `src/safari_zone.c`. The exact story meaning of every `VAR_UNK_4057` value is not yet named by the decomp, so the table deliberately states the proven values and conditions rather than assigning unsupported labels to them.

</details>

Mom's purchase call and the item collection are separate: the queued call announces the purchase, while the home field script retrieves the stored item, checks Bag space, and clears `CALL_TRIGGER_MOM_BOUGHT_SOMETHING` only after a successful award.

---

### Incoming: field-script trigger

Not every incoming call uses the queued trigger-bit table. A normal field script can configure and immediately launch a phone call with these commands. Elm's stolen-Pokemon call is a vanilla direct field-scripted call: it selects `PHONE_SCRIPT_002` and does not set or consume any queued event-trigger bit.

| Opcode | scrcmd / editor name | Source name | Use |
|---|---|---|---|
| `0x01AE` | `SetPhoneCall contact scriptedFlag predefinedScript` | `SetPhoneCall` | Configure a field-scripted phone call. |
| `0x01AF` | `RunPhoneCall` | `RunPhoneCall` | Launch the configured phone call. |

---

## Phone scripts and text archives

### What a phone conversation is made from

The term "phone script" does **not** mean a normal map field script. A normal field script is bytecode that runs on a map, such as the stolen-Pokemon sequence; it can register a number or start a call. A phone conversation is assembled from compact Pokegear data and the contact's message archive.

| Part | What it controls | Where it lives |
|---|---|---|
| Field script | Map events, including direct calls through `0x01AE` and `0x01AF`. | Map script data. |
| Generic header | Whether one ordinary trainer-call outcome is eligible. | Overlay 101. |
| Phone-script definition | The selected message IDs and any result, such as a rematch or gift. | Overlay 101. |
| Contact text archive | Contact name and the actual words printed by messages. | Message NARC. |

The following data flow applies to an ordinary **type-0** contact:

```text
contact and call direction
  -> that contact's outgoing or incoming header rows
  -> first eligible header chooses a phone-script definition ID
  -> definition chooses a message and optional side effect
  -> message ID is read from that contact's text archive
```

This separation matters when editing. A header chooses an outcome; a definition describes that outcome; a text archive supplies its wording.

### Generic phone calls: selection flow

For a type-0 contact, the call code chooses the outgoing or incoming half of the contact's 16 generic-header rows. It checks up to eight rows in order. A matching row supplies a phone-script definition ID. For an outgoing call made while standing on the contact's own `mapId`, the generic handler instead uses the contact record's `phoneScriptIfLocal` value.

The definition selects one message ID for a male player and one for a female player, then optionally applies a result. Known generic results include ordinary dialogue, a pending rematch, a dynamic phone gift, a save-flag operation, and a random inserted word or phrase.

### Generic phone-call data in Overlay 101

The generic tables are compiled into **Overlay 101**, the Pokegear application overlay. They are not in `pmtel_book.dat`, a contact text archive, or a map-script NARC. For the US decompressed `overlay_0101.bin`, Overlay 101 loads at `0x021E7740`.

| Data | US Overlay 101 file range | US RAM range | Addressing rule |
|---|---|---|---|
| Phone-script definitions | `1143C`-`11EEB` | `021F8B7C`-`021F962B` | Definition ID `s`: `1143C + (s * 6)`. IDs `0`-`455`; ID 0 is the all-zero no-script definition. |
| Greeting message IDs | `11EEC`-`11F4B` | `021F962C`-`021F968B` | Eight rows of twelve one-byte message IDs, separate from conversation/result definitions. |
| Generic call headers | `11F4C`-`13B6B` | `021F968C`-`021FB2AB` | Contact ID `c`: `11F4C + (c * 60)`. Each 16-row block is `0x60` bytes. |

All offsets in this section are hexadecimal and apply only to the **US decompressed** Overlay 101. Do not use them in a compressed overlay or another revision without remapping from that overlay's RAM base.

#### Header records: choosing an outcome

Each generic header is six bytes. The first eight rows are outgoing-call choices; the next eight are incoming-call choices.

```text
byte 0: generic condition type
byte 1: chance
bytes 2-3: little-endian header `scriptType` field
bytes 4-5: little-endian phone-script definition ID
```

The decomp names the middle word `scriptType`. It is a playback-handler index: `0`, `1`, and `2` select the simple, species-buffering generic, and random-line handlers; higher values select specialised handlers. It must not be confused with the side-effect type stored in a phone-script definition.

| Direction | Header rows | First-row offset for contact `c` |
|---|---|---|
| Outgoing | `0`-`7` | `11F4C + (c * 60)` |
| Incoming | `8`-`15` | `11F7C + (c * 60)` |

`PHONECALLGENERIC_NIL` (`0`) and `PHONECALLGENERIC_NONE` (`FF`) stop the header scan. They are control rows, not ordinary dialogue outcomes. Do not insert or delete header bytes: later rows are addressed at fixed offsets.

#### Script-definition records: message and side effect

Each phone-script definition is six bytes. Its first two bytes are message IDs in the **current caller's** text archive, selected by the player's gender.

```text
byte 0: message ID for a male player
byte 1: message ID for a female player
bytes 2-3: little-endian word: low 4 bits = side-effect type; upper 12 bits = parameter 0
bytes 4-5: little-endian parameter 1
```

| Low-nibble value | Decomp name | Result |
|:---:|---|---|
| `0` | `NONE` | Prints the selected message with no known persistent phone result. |
| `1` | `UNK1` | Sets or clears a save flag. Its observed implementation is the same as type `2`. |
| `2` | `FLAG` | Sets or clears a save flag. |
| `3` | `REMATCH` | Sets the caller's pending rematch state. |
| `4` | `ITEM` | Queues a dynamic phone gift for collection from the NPC. |
| `5` | `WORD` | Inserts a random word or message fragment. |

The parameter meanings for every individual `ITEM`, `FLAG`, and `WORD` definition are not all documented. Do not alter a non-`NONE` definition's packed fields only because its text looks suitable.

### Contact text archives

Each contact ID maps to one phone message archive through `sPhoneMessageGmm` in `phonebook_dat.c`; Mom, for example, maps to `NARC_msg_msg_0664_bin`. Message **0** in that archive is the contact name displayed by the phone-call UI.

The selected phone-script definition provides the next message ID, which is read from that same contact archive. Therefore one definition ID can produce different wording when used by another contact: its message numbers are reinterpreted against that other contact's archive.

`GetPhoneContactMsgIds` returns the contact archive and an input message number plus one. It does not choose generic versus special dialogue and does not select the caller; those decisions have already been made by the header, special handler, queued-event record, or field script.

### Calls that do not use generic headers

The generic header route is only for type-0 contacts. Mom, Elm, Oak, Gym Leaders, Baoba, and other special contact types use dedicated code to choose their call behaviour. A queued event-call record in Overlay 2 already supplies a predefined phone-script ID, while a field script can configure and launch a direct call through `SetPhoneCall` and `RunPhoneCall`.

Changing a generic header does not change these special, queued, or field-scripted calls.

### Editing existing generic calls

| Goal | Edit | Behavioural risk | Independent test |
|---|---|---|---|
| Change wording only | Edit the selected message in the contact's text archive. | None expected. | Trigger the same call; only the wording changes. |
| Change one incoming outcome | Repoint that incoming header's two-byte definition ID to an existing `NONE` definition. | The replacement definition's message IDs are interpreted in the caller's archive. | Trigger that outcome; other incoming outcomes remain available. |
| Remove a rematch offer | Repoint its header to a known ordinary-dialogue definition, leaving the original definition intact for other users. | Prevents the `seeking` side effect only for that header. | Confirm no map marker or rematch battle appears after the call. |
| Add a definition or condition | Requires verified free Overlay 101 space, a valid new ID, and further code tracing. | Not yet a documented safe binary edit. | Do not treat an appended six-byte record as usable without that work. |

### Advanced reference: code and raw records

<details>
<summary>Click for details...</summary>

The decomp provides these names as a readable source map for the binary logic:

| Decomp function | Source file | Role |
|---|---|---|
| `PhoneCall_GetScriptId_Generic` | `scripts/phone_scripts_generic.c` | Calculates `callerID * 16`, chooses the outgoing or incoming group, and returns the selected definition ID. |
| `PhoneScriptGeneric_GetScriptIdInternal` | `scripts/phone_scripts_generic.c` | Walks at most eight headers, stops at `NIL` or `NONE`, evaluates conditions, and returns a definition ID. |
| `PhoneCall_GetScriptDefPtrByID` | `overlay_101_021F1D74.c` | Resolves `gPhoneCallScriptDef[scriptID]`. |
| `PhoneCall_ApplyGenericNPCcallSideEffect` | `overlay_101_021F1D74.c` | Interprets `REMATCH`, `ITEM`, `UNK1`, `FLAG`, and `WORD` definition side effects. |

The first definition records are `00 00 00 00 00 00` for ID 0, `01 02 00 00 00 00` for `PHONE_SCRIPT_001`, and `21 22 00 00 00 00` for `PHONE_SCRIPT_002`. These first scripts are Elm messages; copying one to another contact can point at different text because the message archive changes.

The first header is `00 64 00 00 00 00`: `NIL`, chance `100`, and definition ID 0. The following unused header is `FF 00 00 00 00 00`: `NONE`.

The compiled function entries for the generic selector and definition interpreter have not yet been independently mapped in US Overlay 101 and should not be guessed from a decomp source filename. The handler-dispatch table itself is mapped: US Overlay 101 file offset `0x10F3C` contains Thumb pointers. Handler indices `0`, `1`, and `2` point to `0x021F2681`, `0x021F2F51`, and `0x021F2FFD` respectively.

</details>

---

## Trainer rematches

### Ordinary phone-trainer rematches

Ordinary trainer rematches use both phone state and a fixed trainer-ID table in the **US vanilla Overlay 26**. In an extracted Overlay 26 file (commonly named `overlay_0026.bin`), the table starts at **file offset `0x20C`**. It has 63 rows of 12 bytes, for a total of `0x2F4` bytes; its occupied range is `0x20C` through `0x4FF` inclusive.

| Item | Binary value |
|---|---|
| Row count | `63` (`0x3F`) |
| Row size | `12` bytes (`0x0C`) |
| Entry type | Six little-endian `u16` trainer battle IDs |
| Row `r` offset | `0x20C + (r * 0x0C)` |
| Entry `e` within row | `rowOffset + (e * 2)` |

The six entries are:

| Entry | Relative offset | Meaning |
|:---:|:---:|---|
| `0` | `00` | Original/base battle ID. Normally not selected, but it can be returned as a fallback; keep it valid. |
| `1` | `02` | Duplicate base battle ID: first selectable entry before rematch groups are unlocked. |
| `2` | `04` | Rematch battle 1. |
| `3` | `06` | Rematch battle 2. |
| `4` | `08` | Rematch battle 3. |
| `5` | `0A` | Rematch battle 4, or `TRAINER_NONE` (`0x0000`). |

The normal flow is:

1. A phone script sets `PhoneRematch.seeking` for the contact.
2. The trainer battle lookup checks that bit and uses the contact's `trainerId` to find its row.
3. It tests entries from index `1` through `5`, selecting the first unbeaten entry whose rematch-group requirements are unlocked.

The initial scan begins at index `1`, so index `1` is normally the first rematch battle. However, index `0` is a real return value, not an unreachable dummy: it is returned when the scan encounters `0` at index `1`, and it can also be the fallback when a later rematch group is not unlocked.  

:::caution
`0xFFFF` is skipped only while the rematch code searches for an unbeaten candidate or walks backwards for an unlocked fallback. It is not skipped when the scan sees a `0` terminator and returns the preceding index, and the resulting trainer ID is not validated. A malformed custom layout can therefore return literal trainer ID `0xFFFF`; do not use `0xFFFF` as a repeat marker or immediately before a `0` terminator.
:::

:::warning
`0x20C` is a **US Overlay 26 file offset**, not an ARM9 address and not an offset in `pmtel_book.dat`. It must be independently located for another game revision before editing. Do not insert or delete bytes: the game indexes fixed 12-byte rows.
:::

### Vanilla US Overlay 26 table

The following binary-table export is from local community research supplied by **Eclipse (Luna)**, cross-checked against the HeartGold trainer-ID constants. Each cell gives the trainer battle ID followed by its decomp label. Write the ID as a little-endian `u16` in the ROM: for example, ID `151` is `0x0097`, or bytes `97 00`.

<details>
<summary>Show all 63 rows</summary>

| Row | Entry 0: base | Entry 1: duplicate base | Entry 2 | Entry 3 | Entry 4 | Entry 5 |
|:---:|---|---|---|---|---|---|
| 0 | `414` (`TRAINER_BIRD_KEEPER_GS_JOSE_2`) | `414` (`TRAINER_BIRD_KEEPER_GS_JOSE_2`) | `303` (`TRAINER_BIRD_KEEPER_GS_JOSE`) | `446` (`TRAINER_BIRD_KEEPER_GS_JOSE_3`) | `602` (`TRAINER_BIRD_KEEPER_GS_JOSE_4`) | `0` (`TRAINER_NONE`) |
| 1 | `151` (`TRAINER_PICNICKER_ERIN`) | `151` (`TRAINER_PICNICKER_ERIN`) | `335` (`TRAINER_PICNICKER_ERIN_2`) | `453` (`TRAINER_PICNICKER_ERIN_3`) | `603` (`TRAINER_PICNICKER_ERIN_4`) | `0` (`TRAINER_NONE`) |
| 2 | `27` (`TRAINER_PICNICKER_LIZ`) | `27` (`TRAINER_PICNICKER_LIZ`) | `276` (`TRAINER_PICNICKER_LIZ_2`) | `277` (`TRAINER_PICNICKER_LIZ_3`) | `518` (`TRAINER_PICNICKER_LIZ_4`) | `0` (`TRAINER_NONE`) |
| 3 | `397` (`TRAINER_SCHOOL_KID_M_CHAD`) | `397` (`TRAINER_SCHOOL_KID_M_CHAD`) | `434` (`TRAINER_SCHOOL_KID_M_CHAD_2`) | `435` (`TRAINER_SCHOOL_KID_M_CHAD_3`) | `507` (`TRAINER_SCHOOL_KID_M_CHAD_4`) | `0` (`TRAINER_NONE`) |
| 4 | `211` (`TRAINER_SAILOR_HUEY`) | `211` (`TRAINER_SAILOR_HUEY`) | `440` (`TRAINER_SAILOR_HUEY_2`) | `441` (`TRAINER_SAILOR_HUEY_3`) | `509` (`TRAINER_SAILOR_HUEY_4`) | `0` (`TRAINER_NONE`) |
| 5 | `4` (`TRAINER_BUG_CATCHER_WADE`) | `4` (`TRAINER_BUG_CATCHER_WADE`) | `461` (`TRAINER_BUG_CATCHER_WADE_3`) | `460` (`TRAINER_BUG_CATCHER_WADE_2`) | `512` (`TRAINER_BUG_CATCHER_WADE_4`) | `0` (`TRAINER_NONE`) |
| 6 | `8` (`TRAINER_YOUNGSTER_JOEY`) | `8` (`TRAINER_YOUNGSTER_JOEY`) | `279` (`TRAINER_YOUNGSTER_JOEY_2`) | `280` (`TRAINER_YOUNGSTER_JOEY_3`) | `510` (`TRAINER_YOUNGSTER_JOEY_4`) | `0` (`TRAINER_NONE`) |
| 7 | `178` (`TRAINER_SCHOOL_KID_M_JACK`) | `178` (`TRAINER_SCHOOL_KID_M_JACK`) | `430` (`TRAINER_SCHOOL_KID_M_JACK_2`) | `431` (`TRAINER_SCHOOL_KID_M_JACK_3`) | `503` (`TRAINER_SCHOOL_KID_M_JACK_4`) | `0` (`TRAINER_NONE`) |
| 8 | `102` (`TRAINER_ACE_TRAINER_M_GAVEN`) | `102` (`TRAINER_ACE_TRAINER_M_GAVEN`) | `456` (`TRAINER_ACE_TRAINER_M_GAVEN_2`) | `457` (`TRAINER_ACE_TRAINER_M_GAVEN_3`) | `604` (`TRAINER_ACE_TRAINER_M_GAVEN_4`) | `0` (`TRAINER_NONE`) |
| 9 | `17` (`TRAINER_BLACK_BELT_KENJI`) | `17` (`TRAINER_BLACK_BELT_KENJI`) | `250` (`TRAINER_BLACK_BELT_KENJI_2`) | `278` (`TRAINER_BLACK_BELT_KENJI_3`) | `605` (`TRAINER_BLACK_BELT_KENJI_4`) | `0` (`TRAINER_NONE`) |
| 10 | `145` (`TRAINER_HIKER_PARRY`) | `145` (`TRAINER_HIKER_PARRY`) | `451` (`TRAINER_HIKER_PARRY_2`) | `452` (`TRAINER_HIKER_PARRY_3`) | `606` (`TRAINER_HIKER_PARRY_4`) | `0` (`TRAINER_NONE`) |
| 11 | `402` (`TRAINER_PICNICKER_TIFFANY`) | `402` (`TRAINER_PICNICKER_TIFFANY`) | `466` (`TRAINER_PICNICKER_TIFFANY_2`) | `467` (`TRAINER_PICNICKER_TIFFANY_3`) | `522` (`TRAINER_PICNICKER_TIFFANY_4`) | `0` (`TRAINER_NONE`) |
| 12 | `61` (`TRAINER_HIKER_ANTHONY`) | `61` (`TRAINER_HIKER_ANTHONY`) | `100` (`TRAINER_HIKER_ANTHONY_2`) | `155` (`TRAINER_HIKER_ANTHONY_3`) | `523` (`TRAINER_HIKER_ANTHONY_4`) | `0` (`TRAINER_NONE`) |
| 13 | `114` (`TRAINER_ACE_TRAINER_F_REENA`) | `114` (`TRAINER_ACE_TRAINER_F_REENA`) | `444` (`TRAINER_ACE_TRAINER_F_REENA_2`) | `445` (`TRAINER_ACE_TRAINER_F_REENA_3`) | `607` (`TRAINER_ACE_TRAINER_F_REENA_4`) | `0` (`TRAINER_NONE`) |
| 14 | `124` (`TRAINER_FISHERMAN_WILTON`) | `124` (`TRAINER_FISHERMAN_WILTON`) | `325` (`TRAINER_FISHERMAN_WILTON_2`) | `450` (`TRAINER_FISHERMAN_WILTON_3`) | `608` (`TRAINER_FISHERMAN_WILTON_4`) | `0` (`TRAINER_NONE`) |
| 15 | `113` (`TRAINER_ACE_TRAINER_F_JAMIE`) | `113` (`TRAINER_ACE_TRAINER_F_JAMIE`) | `458` (`TRAINER_ACE_TRAINER_F_JAMIE_2`) | `459` (`TRAINER_ACE_TRAINER_F_JAMIE_3`) | `609` (`TRAINER_ACE_TRAINER_F_JAMIE_4`) | `0` (`TRAINER_NONE`) |
| 16 | `7` (`TRAINER_JUGGLER_IRWIN`) | `7` (`TRAINER_JUGGLER_IRWIN`) | `454` (`TRAINER_JUGGLER_IRWIN_2`) | `455` (`TRAINER_JUGGLER_IRWIN_3`) | `527` (`TRAINER_JUGGLER_IRWIN_4`) | `0` (`TRAINER_NONE`) |
| 17 | `131` (`TRAINER_POKE_MANIAC_BRENT`) | `131` (`TRAINER_POKE_MANIAC_BRENT`) | `172` (`TRAINER_POKE_MANIAC_BRENT_2`) | `173` (`TRAINER_POKE_MANIAC_BRENT_3`) | `530` (`TRAINER_POKE_MANIAC_BRENT_4`) | `0` (`TRAINER_NONE`) |
| 18 | `24` (`TRAINER_SCHOOL_KID_M_ALAN`) | `24` (`TRAINER_SCHOOL_KID_M_ALAN`) | `432` (`TRAINER_SCHOOL_KID_M_ALAN_2`) | `433` (`TRAINER_SCHOOL_KID_M_ALAN_3`) | `505` (`TRAINER_SCHOOL_KID_M_ALAN_4`) | `0` (`TRAINER_NONE`) |
| 19 | `44` (`TRAINER_POKEFAN_M_DEREK`) | `44` (`TRAINER_POKEFAN_M_DEREK`) | `438` (`TRAINER_POKEFAN_M_DEREK_2`) | `439` (`TRAINER_POKEFAN_M_DEREK_3`) | `610` (`TRAINER_POKEFAN_M_DEREK_4`) | `0` (`TRAINER_NONE`) |
| 20 | `65` (`TRAINER_PICNICKER_GINA`) | `65` (`TRAINER_PICNICKER_GINA`) | `142` (`TRAINER_PICNICKER_GINA_2`) | `334` (`TRAINER_PICNICKER_GINA_3`) | `520` (`TRAINER_PICNICKER_GINA_4`) | `0` (`TRAINER_NONE`) |
| 21 | `123` (`TRAINER_FISHERMAN_TULLY`) | `123` (`TRAINER_FISHERMAN_TULLY`) | `323` (`TRAINER_FISHERMAN_TULLY_2`) | `324` (`TRAINER_FISHERMAN_TULLY_3`) | `517` (`TRAINER_FISHERMAN_TULLY_4`) | `0` (`TRAINER_NONE`) |
| 22 | `182` (`TRAINER_POKEFAN_BEVERLY`) | `182` (`TRAINER_POKEFAN_BEVERLY`) | `436` (`TRAINER_POKEFAN_BEVERLY_2`) | `437` (`TRAINER_POKEFAN_BEVERLY_3`) | `611` (`TRAINER_POKEFAN_BEVERLY_4`) | `0` (`TRAINER_NONE`) |
| 23 | `137` (`TRAINER_BIRD_KEEPER_GS_VANCE`) | `137` (`TRAINER_BIRD_KEEPER_GS_VANCE`) | `447` (`TRAINER_BIRD_KEEPER_GS_VANCE_2`) | `448` (`TRAINER_BIRD_KEEPER_GS_VANCE_3`) | `612` (`TRAINER_BIRD_KEEPER_GS_VANCE_4`) | `0` (`TRAINER_NONE`) |
| 24 | `57` (`TRAINER_FISHERMAN_RALPH`) | `57` (`TRAINER_FISHERMAN_RALPH`) | `462` (`TRAINER_FISHERMAN_RALPH_2`) | `463` (`TRAINER_FISHERMAN_RALPH_3`) | `515` (`TRAINER_FISHERMAN_RALPH_4`) | `0` (`TRAINER_NONE`) |
| 25 | `66` (`TRAINER_CAMPER_TODD`) | `66` (`TRAINER_CAMPER_TODD`) | `274` (`TRAINER_CAMPER_TODD_2`) | `275` (`TRAINER_CAMPER_TODD_3`) | `525` (`TRAINER_CAMPER_TODD_4`) | `0` (`TRAINER_NONE`) |
| 26 | `78` (`TRAINER_BUG_CATCHER_ARNIE`) | `78` (`TRAINER_BUG_CATCHER_ARNIE`) | `360` (`TRAINER_BUG_CATCHER_ARNIE_2`) | `449` (`TRAINER_BUG_CATCHER_ARNIE_3`) | `513` (`TRAINER_BUG_CATCHER_ARNIE_4`) | `0` (`TRAINER_NONE`) |
| 27 | `400` (`TRAINER_LASS_DANA`) | `400` (`TRAINER_LASS_DANA`) | `464` (`TRAINER_LASS_DANA_2`) | `465` (`TRAINER_LASS_DANA_3`) | `528` (`TRAINER_LASS_DANA_4`) | `0` (`TRAINER_NONE`) |
| 28 | `184` (`TRAINER_LASS_KRISE`) | `184` (`TRAINER_LASS_KRISE`) | `613` (`TRAINER_LASS_KRISE_2`) | `614` (`TRAINER_LASS_KRISE_3`) | `615` (`TRAINER_LASS_KRISE_4`) | `0` (`TRAINER_NONE`) |
| 29 | `64` (`TRAINER_YOUNGSTER_IAN`) | `64` (`TRAINER_YOUNGSTER_IAN`) | `616` (`TRAINER_YOUNGSTER_IAN_2`) | `617` (`TRAINER_YOUNGSTER_IAN_3`) | `618` (`TRAINER_YOUNGSTER_IAN_4`) | `0` (`TRAINER_NONE`) |
| 30 | `388` (`TRAINER_FIREBREATHER_WALT`) | `388` (`TRAINER_FIREBREATHER_WALT`) | `619` (`TRAINER_FIREBREATHER_WALT_2`) | `620` (`TRAINER_FIREBREATHER_WALT_3`) | `621` (`TRAINER_FIREBREATHER_WALT_4`) | `0` (`TRAINER_NONE`) |
| 31 | `140` (`TRAINER_BUG_CATCHER_DOUG`) | `140` (`TRAINER_BUG_CATCHER_DOUG`) | `622` (`TRAINER_BUG_CATCHER_DOUG_2`) | `623` (`TRAINER_BUG_CATCHER_DOUG_3`) | `624` (`TRAINER_BUG_CATCHER_DOUG_4`) | `0` (`TRAINER_NONE`) |
| 32 | `48` (`TRAINER_BUG_CATCHER_ROB`) | `48` (`TRAINER_BUG_CATCHER_ROB`) | `625` (`TRAINER_BUG_CATCHER_ROB_2`) | `626` (`TRAINER_BUG_CATCHER_ROB_3`) | `627` (`TRAINER_BUG_CATCHER_ROB_4`) | `0` (`TRAINER_NONE`) |
| 33 | `313` (`TRAINER_BIKER_REESE`) | `313` (`TRAINER_BIKER_REESE`) | `628` (`TRAINER_BIKER_REESE_2`) | `629` (`TRAINER_BIKER_REESE_3`) | `630` (`TRAINER_BIKER_REESE_4`) | `0` (`TRAINER_NONE`) |
| 34 | `574` (`TRAINER_BIKER_AIDEN`) | `574` (`TRAINER_BIKER_AIDEN`) | `631` (`TRAINER_BIKER_AIDEN_2`) | `632` (`TRAINER_BIKER_AIDEN_3`) | `633` (`TRAINER_BIKER_AIDEN_4`) | `0` (`TRAINER_NONE`) |
| 35 | `579` (`TRAINER_BIKER_ERNEST`) | `579` (`TRAINER_BIKER_ERNEST`) | `634` (`TRAINER_BIKER_ERNEST_2`) | `635` (`TRAINER_BIKER_ERNEST_3`) | `636` (`TRAINER_BIKER_ERNEST_4`) | `0` (`TRAINER_NONE`) |
| 36 | `382` (`TRAINER_TEACHER_HILLARY`) | `382` (`TRAINER_TEACHER_HILLARY`) | `637` (`TRAINER_TEACHER_HILLARY_2`) | `638` (`TRAINER_TEACHER_HILLARY_3`) | `639` (`TRAINER_TEACHER_HILLARY_4`) | `0` (`TRAINER_NONE`) |
| 37 | `331` (`TRAINER_SCHOOL_KID_M_BILLY`) | `331` (`TRAINER_SCHOOL_KID_M_BILLY`) | `640` (`TRAINER_SCHOOL_KID_M_BILLY_2`) | `641` (`TRAINER_SCHOOL_KID_M_BILLY_3`) | `642` (`TRAINER_SCHOOL_KID_M_BILLY_4`) | `0` (`TRAINER_NONE`) |
| 38 | `569` (`TRAINER_TWINS_KAY_AND_TIA`) | `569` (`TRAINER_TWINS_KAY_AND_TIA`) | `643` (`TRAINER_TWINS_KAY_AND_TIA_2`) | `644` (`TRAINER_TWINS_KAY_AND_TIA_3`) | `645` (`TRAINER_TWINS_KAY_AND_TIA_4`) | `0` (`TRAINER_NONE`) |
| 39 | `565` (`TRAINER_BIRD_KEEPER_GS_JOSH`) | `565` (`TRAINER_BIRD_KEEPER_GS_JOSH`) | `646` (`TRAINER_BIRD_KEEPER_GS_JOSH_2`) | `647` (`TRAINER_BIRD_KEEPER_GS_JOSH_3`) | `648` (`TRAINER_BIRD_KEEPER_GS_JOSH_4`) | `0` (`TRAINER_NONE`) |
| 40 | `567` (`TRAINER_SCHOOL_KID_M_TORIN`) | `567` (`TRAINER_SCHOOL_KID_M_TORIN`) | `649` (`TRAINER_SCHOOL_KID_M_TORIN_2`) | `650` (`TRAINER_SCHOOL_KID_M_TORIN_3`) | `651` (`TRAINER_SCHOOL_KID_M_TORIN_4`) | `0` (`TRAINER_NONE`) |
| 41 | `559` (`TRAINER_YOUNG_COUPLE_TIM_AND_SUE`) | `559` (`TRAINER_YOUNG_COUPLE_TIM_AND_SUE`) | `652` (`TRAINER_YOUNG_COUPLE_TIM_AND_SUE_2`) | `653` (`TRAINER_YOUNG_COUPLE_TIM_AND_SUE_3`) | `654` (`TRAINER_YOUNG_COUPLE_TIM_AND_SUE_4`) | `0` (`TRAINER_NONE`) |
| 42 | `358` (`TRAINER_HIKER_KENNY`) | `358` (`TRAINER_HIKER_KENNY`) | `655` (`TRAINER_HIKER_KENNY_2`) | `656` (`TRAINER_HIKER_KENNY_3`) | `657` (`TRAINER_HIKER_KENNY_4`) | `0` (`TRAINER_NONE`) |
| 43 | `561` (`TRAINER_CAMPER_TANNER`) | `561` (`TRAINER_CAMPER_TANNER`) | `658` (`TRAINER_CAMPER_TANNER_2`) | `659` (`TRAINER_CAMPER_TANNER_3`) | `660` (`TRAINER_CAMPER_TANNER_4`) | `0` (`TRAINER_NONE`) |
| 44 | `59` (`TRAINER_FISHERMAN_KYLE`) | `59` (`TRAINER_FISHERMAN_KYLE`) | `661` (`TRAINER_FISHERMAN_KYLE_2`) | `662` (`TRAINER_FISHERMAN_KYLE_3`) | `663` (`TRAINER_FISHERMAN_KYLE_4`) | `0` (`TRAINER_NONE`) |
| 45 | `558` (`TRAINER_FISHERMAN_KYLER`) | `558` (`TRAINER_FISHERMAN_KYLER`) | `664` (`TRAINER_FISHERMAN_KYLER_2`) | `665` (`TRAINER_FISHERMAN_KYLER_3`) | `666` (`TRAINER_FISHERMAN_KYLER_4`) | `0` (`TRAINER_NONE`) |
| 46 | `401` (`TRAINER_GENTLEMAN_ALFRED`) | `401` (`TRAINER_GENTLEMAN_ALFRED`) | `672` (`TRAINER_GENTLEMAN_ALFRED_2`) | `673` (`TRAINER_GENTLEMAN_ALFRED_3`) | `674` (`TRAINER_GENTLEMAN_ALFRED_4`) | `0` (`TRAINER_NONE`) |
| 47 | `20` (`TRAINER_LEADER_FALKNER_FALKNER`) | `20` (`TRAINER_LEADER_FALKNER_FALKNER`) | `712` (`TRAINER_LEADER_FALKNER_FALKNER_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 48 | `21` (`TRAINER_LEADER_BUGSY_BUGSY`) | `21` (`TRAINER_LEADER_BUGSY_BUGSY`) | `713` (`TRAINER_LEADER_BUGSY_BUGSY_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 49 | `30` (`TRAINER_LEADER_WHITNEY`) | `30` (`TRAINER_LEADER_WHITNEY`) | `714` (`TRAINER_LEADER_WHITNEY_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 50 | `31` (`TRAINER_LEADER_MORTY_MORTY`) | `31` (`TRAINER_LEADER_MORTY_MORTY`) | `715` (`TRAINER_LEADER_MORTY_MORTY_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 51 | `33` (`TRAINER_LEADER_JASMINE_JASMINE`) | `33` (`TRAINER_LEADER_JASMINE_JASMINE`) | `717` (`TRAINER_LEADER_JASMINE_JASMINE_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 52 | `34` (`TRAINER_LEADER_CHUCK_CHUCK`) | `34` (`TRAINER_LEADER_CHUCK_CHUCK`) | `718` (`TRAINER_LEADER_CHUCK_CHUCK_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 53 | `32` (`TRAINER_LEADER_PRYCE_PRYCE`) | `32` (`TRAINER_LEADER_PRYCE_PRYCE`) | `716` (`TRAINER_LEADER_PRYCE_PRYCE_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 54 | `35` (`TRAINER_LEADER_CLAIR_CLAIR`) | `35` (`TRAINER_LEADER_CLAIR_CLAIR`) | `719` (`TRAINER_LEADER_CLAIR_CLAIR_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 55 | `253` (`TRAINER_LEADER_BROCK_BROCK`) | `253` (`TRAINER_LEADER_BROCK_BROCK`) | `720` (`TRAINER_LEADER_BROCK_BROCK_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 56 | `254` (`TRAINER_LEADER_MISTY_MISTY`) | `254` (`TRAINER_LEADER_MISTY_MISTY`) | `721` (`TRAINER_LEADER_MISTY_MISTY_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 57 | `255` (`TRAINER_LEADER_LT_SURGE_LT__SURGE`) | `255` (`TRAINER_LEADER_LT_SURGE_LT__SURGE`) | `722` (`TRAINER_LEADER_LT_SURGE_LT__SURGE_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 58 | `256` (`TRAINER_LEADER_ERIKA_ERIKA`) | `256` (`TRAINER_LEADER_ERIKA_ERIKA`) | `723` (`TRAINER_LEADER_ERIKA_ERIKA_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 59 | `257` (`TRAINER_LEADER_JANINE_JANINE`) | `257` (`TRAINER_LEADER_JANINE_JANINE`) | `724` (`TRAINER_LEADER_JANINE_JANINE_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 60 | `258` (`TRAINER_LEADER_SABRINA_SABRINA`) | `258` (`TRAINER_LEADER_SABRINA_SABRINA`) | `725` (`TRAINER_LEADER_SABRINA_SABRINA_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 61 | `259` (`TRAINER_LEADER_BLAINE_BLAINE`) | `259` (`TRAINER_LEADER_BLAINE_BLAINE`) | `726` (`TRAINER_LEADER_BLAINE_BLAINE_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |
| 62 | `261` (`TRAINER_LEADER_BLUE_BLUE`) | `261` (`TRAINER_LEADER_BLUE_BLUE`) | `727` (`TRAINER_LEADER_BLUE_BLUE_2`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) | `0` (`TRAINER_NONE`) |

</details>

### Schedule and story gates

The weekday/time fields are requirements only for a phone-script condition that reads them; they are not universal rematch requirements.

For ordinary generic contacts, the header that creates the scheduled rematch offer is `PHONECALLGENERIC_POSTROCKETWEEKTIME`. It requires all of the following:

1. `FLAG_BEAT_RADIO_TOWER_ROCKETS` is set.
2. The hardware weekday equals `rematchWeekday`.
3. The current morning/day/night value equals `rematchTimeOfDay`.
4. The header's chance roll succeeds.

The generic `FIGHTPOSTROCKETS` condition also prevents a generic rematch script from being selected before the Radio Tower flag is set. Unlike `POSTROCKETWEEKTIME`, it does not read the phonebook record's weekday or time bytes, so an offer using this condition is not schedule-gated. Other headers prevent another offer while `PhoneRematch.seeking` is already set, prevent calls while a dynamic gift is waiting, and exclude National Park during the Bug-Catching Contest. These are separate header conditions, so changing only the two schedule bytes does not remove the story and persistent-state gates.

Gym Leaders use the same two schedule bytes in their dedicated handler, but their requirements are different: the player must have all 16 badges, no rematch may already be pending for that leader, the schedule must match, and the Fighting Dojo must have capacity before the leader sets `seeking`.

### Gym Leader rematches

Gym Leaders do not use the ordinary overlay 26 trainer-progression table. Their dedicated phone handler uses persistent rematch state and Fighting Dojo availability to offer the Gym Leader rematch flow.

Changing an ordinary trainer's overlay 26 row cannot create a Gym Leader rematch. Likewise, changing a Gym Leader's weekday or normal trainer ID does not replace the Fighting Dojo logic.

---

## Gifts after calls and rematches

There are two item systems that are easy to confuse:

| Gift path | Storage | Retrieval |
|---|---|---|
| Contact-configured rematch reward | `PhoneBookEntry.gift` | The post-rematch battle field script clears the pending rematch, calls `GetPhoneContactRandomGiftBerry`, and awards the returned item if it is nonzero. `ITEM_CHERI_BERRY` is a special sentinel: it returns one of the first ten berries at random. |
| Dynamic phone-queued gift | `PhoneRematch.giftItem` | A `PHONESCRIPTTYPE_ITEM` phone outcome stores an item here. The field script calls `GetPhoneContactGiftItem` when no rematch is active; that command clears the queued item after reading it. |

The first path is not a generic “static item” given merely because a contact exists. In vanilla it is the reward path after a trainer's rematch battle. The nonzero rows are Joey (HP Up), Kenji (PP Up), Huey (Protein), Vance (Carbos), Parry (Iron), Erin (Calcium), and Ian (random berry via the Cheri sentinel).

The second path is the phone-only item offer system. Because the NPC field script checks for an available rematch before calling `GetPhoneContactGiftItem`, a queued dynamic gift is collected when no rematch battle is currently selected. This is the path relevant to item-only phone outcomes.

---

## Hex editing notes

<details>
<summary>Editing <code>pmtel_book.dat</code> directly</summary>

1. Extract or open the direct NitroFS file `tel/pmtel_book.dat`.
2. Read the 32-bit little-endian count at `0x00`.
3. Contact record `n` starts at `0x04 + (n * 0x14)`.
4. Use the table in [Phonebook data](#phonebook-data-pmtel_bookdat) for field offsets.
5. Keep the record count and record size unchanged unless you also understand every contact-ID-indexed table in the game.

For example, record-relative offset `0x0A` is the two-byte local generic outgoing script ID. Record-relative offset `0x0F` is the random-call bucket. All multi-byte values are little-endian.

Do not edit unknown fields, and do not add a new contact merely by appending a record: message archive mappings, persistent-state array indexes, contact constants, and other ID-indexed tables must also be expanded.

</details>

---

## See also

- [pret/pokeheartgold: phonebook reader and message mapping](https://github.com/pret/pokeheartgold/blob/master/src/phonebook_dat.c)
- [pret/pokeheartgold: phone contact structure](https://github.com/pret/pokeheartgold/blob/master/include/gear_phone.h)
- [pret/pokeheartgold: incoming call manager and event table](https://github.com/pret/pokeheartgold/blob/master/src/field/overlay_2_gear_phone.c)
- [pret/pokeheartgold: call direction/type dispatch](https://github.com/pret/pokeheartgold/blob/master/src/application/pokegear/phone/overlay_101_021F1D74.c)
- [pret/pokeheartgold: generic phone conditions](https://github.com/pret/pokeheartgold/blob/master/src/application/pokegear/phone/scripts/phone_scripts_generic.c)
- [pret/pokeheartgold: Elm handler](https://github.com/pret/pokeheartgold/blob/master/src/application/pokegear/phone/scripts/phone_scripts_prof_elm.c)
- [pret/pokeheartgold: Mom handler](https://github.com/pret/pokeheartgold/blob/master/src/application/pokegear/phone/scripts/phone_scripts_mother.c)
- [pret/pokeheartgold: call trigger constants](https://github.com/pret/pokeheartgold/blob/master/include/constants/phone_constants.h)
- [pret/pokeheartgold: map-header phone permissions](https://github.com/pret/pokeheartgold/blob/master/include/map_header.h)
- [pret/pokeheartgold: ordinary trainer rematches](https://github.com/pret/pokeheartgold/blob/master/src/overlay_26_022598C0.c)
- [DSPRE HGSS script command reference](https://github.com/DS-Pokemon-Rom-Editor/DSPRE/blob/master/DS_Map/Resources/HGSSCommands.md) - `0x0092` and `0x0093` use the legacy/editor command names shown above.
- [HGSS script command and ARM9 offset research](https://projectpokemon.org/home/forums/topic/54137-hgss-list-of-scripting-commands-and-arm9-offsets/)
- [Project Pokemon: saved Pokegear contact research](https://projectpokemon.org/home/forums/topic/49014-hgss-pokegear-phone-numbers-memory-location/)

<!-- PUBLISHING BLOCKER (Liam only): Submit an HGSS scrcmd-reference update before publishing. Define `0x0094` / `ScrCmd_148` as `SetPhoneCallTrigger triggerId, expedite` (two literal bytes; trigger IDs 0-12; nonzero expedite advances the shared incoming-call timer to threshold - 1), and define `0x0095` as `UnsetPhoneCallTrigger triggerId` (one literal byte; trigger IDs 0-12). -->
