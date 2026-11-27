---
title: Overworld Sprite Replacement Guide
tags:
  - Guide (Diamond)
  - Guide (Pearl)
  - Guide (Platinum)
  - Guide (HeartGold)
  - Guide (SoulSilver)
---

# Overworld Sprite Replacement Guide
> Author(s): [SpagoAsparago](https://github.com/SpagoAsparago) <br />
> Research: [Known Sprite Locations document](https://docs.google.com/document/d/1_nRfhDEoNFbvYP-yjx4oAWmgGXxvqFBvLwYANFehxUU/edit), BagBoy (OW tables research).

This is a guide on how to extract, replace, change the properties and make your own overworld sprites.
The necessary tools are Tinke 0.9.2 and BTX Editor 2.0.

In addition for the Sprite Properties section you will need DSPRE and an Hex Editor like HxD as well.

---
 ## Table of Contents
* [Extracting the Sprite](#extracting-the-sprite)
* [Making your own Sprite](#making-your-own-sprite)
* [Replacing the Sprite](#replacing-the-sprite)
  * [Unused Sprites](#unused-sprites)
* [Changing the Sprite Properties](#changing-the-sprite-properties)
  * [Platinum](#platinum)
  * [HGSS](#hgss)

## Extracting the Sprite  
1. In DSPRE open the ROM to be edited.
2. Navigate to the **Overworld Editor** from the **Other Editors** menu option.
3. Select the overworld to export (by it's OW Entry ID), and click `Export PNG`.

![](resources/dspre_overworld_btx_editor.png)

<details>
<summary>Legacy Tinke BTX export method</summary>
	
Overworld sprites are contained in BTX files, start by opening your ROM with Tinke and then navigating to the following filepath depending on your game:
* **DPPt**: mmodel/mmodel.narc
* **HGSS**: a/0/8/1

Once you've navigated to this directory in Tinke, select the narc file and click unpack to see see all the BTX files.
You can select one and click the "View" button to see the spritesheet. Select the one you want to replace and click Extract, so that you will obtain the BTX file.

![](resources/tinke.PNG)

You can load this file in the BTX Editor 2.0 to either extract the spritesheet itself in a .png format or import your own spritesheet.

![](resources/btxeditor.PNG)

<details>
 <summary>Where to find all Player Character Sprites.</summary>
 <p>
   Diamond and Pearl
   Lucas:
		90 - Walking/Running
		92 - Bike
		155 - Pull out Pokeball
		157 - Water flower
		159 - Surf
164 - Tux
166 - Fishing
363 - Aerobics
365 - Save
367 - Pokecenter

Heroine:
91 - Walking/Running
		93 - Bike
		156 - Pull out Pokeball
		158 - Water flower
		160 - Surf
165 - Dress
167 - Fishing
364 - Aerobics
366 - Save
368 - Pokecenter

  </p>
</details>

</details>

## Making your own Sprite
Before proceeding you should be familiar with Sprite Indexing, as all the sprites must have at most 16 color, with the background color being indexed to the first slot of the palette.
See [Sprite Indexing](/docs/universal/guides/sprite_indexing/) for more information.

First you want to load the png spritesheet you previously extracted in the graphic editing software of your choice, to either make edits to it or use it as a base for your spritesheet.
You may also want to enable some kind of grid visualization in order to view the proper position of each frame. In Aseprite you can do so by clicking View>Grid>Grid Settings.

If you're making your own sprite from scratch, remember that all the frames you draw must be in the same position as the ones in the original sprite, otherwise the movement in game will look wrong; for example the following image is not a correct spritesheet since the three walking frames aren't in the same position as the original one:

![](resources/wrongsprite.PNG)

Once you're done, you can open the BTX you previously extracted in BTX Editor 2.0 and click import, then import the spritesheet you made, and save the BTX.

## Replacing the Sprite  
:::info
If your replacement sprite has a different size and/or number of frames from the sprite to the original (and you will therefore need to change the [sprite properties](#changing-the-sprite-properties) as well, the Tinke method will be needed. If the replacement sprite is the same size and number of frames, the DSPRE import method can be used.
:::

### Tinke BTX Import Method  
Replacing the sprite simply consists in just clicking the "Change File" button in Tinke on the BTX file you want to replace, and then select the BTX you either extracted or obtained from BTX Editor. Don't forget to **click the Pack button on the narc file** after you've replaced the BTX. Now you can save the ROM.

Overworld sprites are contained in BTX files, start by opening your ROM with Tinke and then navigating to the following filepath depending on your game:
* **DPPt**: mmodel/mmodel.narc
* **HGSS**: a/0/8/1

Just remember that the sprite you're inserting must have the same size and number of frames as the one you're replacing, otherwise you will have to change its properties to get the movement working correctly even if it's the same size. Single-frame sprites with the same dimension as the original won't have the issue however.

### DSPRE Import Method  
1. Return to DSPRE.
2. Navigate back to the **Overworld Editor** from the **Other Editors** menu option.
3. Select the overworld to override (by it's OW Entry ID), and click `Import PNG`.

DSPRE will only accept a replacement sprite sheet of the same dimensions as the original, if you need to change the size or number of frames, use the Tinke method.

### Unused Sprites
Adding new overworld sprites would require ASM but luckily there are a number of unused sprites that may be useful to you:

![](resources/unusedsprites.png)

## Changing the Sprite Properties
If you've replaced a sprite with one having a different number of frames or a different size, the sprite will either not display correctly in game or not display at all if it has a different size.
Unless your sprite is only made of one frame and has the same size the original one, you will need to fix this by hex editing each sprite properties. If you haven't already open your ROM in DSPRE so you can easily access the necessary overlay files in the folder created by DSPRE, which will be named `YourROMName_DSPRE_Contents/arm9_overlays/`.

All the offsets listed are based on the US version of the ROM and it has not been tested on other versions.

### Platinum
In Platinum there are two tables we need to look up for changing sprite properties, both are in `Overlay 5`.

First we need to find the ID number of the sprite we previously replaced. For my example I'll use the unused Rotom Oven, whose model number is 406, and I have replaced it with a 64x64 Ho-Oh Sprite ripped from HGSS, having two frames:

![](resources/hooh64x64.png)

The first table is at `0x2BC34` and will look like this:

![](resources/pt_owtable1.PNG)

Each entry is made of 8 bytes, the first 4 in red for the ID number and the latter 4 in blue for the mmodel number. We already know the model number, which in little endian is `96 01`, so we just need to search those bytes using HxD search hex values function with the search direction being set to "forward", then click Search All.
Since my mmodel number is greater than 255, I'll get a single result, being the two bytes corresponding to the model number, the next 2 bytes are going to be 00 00 and the next 4 bytes are the ID number we are looking for, in my case `F4 00 00 00`.

If your model number was less than or equal to 255, **you need to look at the second result in the results tab**.

With this information we can go to the second table at `0x2CA08`, which is structured like this:

![](resources/pt_owtable2.PNG)

Each row is an entry, the bytes in the red columns are the ID numbers of each sprites and the bytes in green are the sprite properties. You can scroll down until you see the ID number from the 1st table in the red columns. The bytes you want to edit are right before the ID number, in my case it will be which is a standard 32x32 16 frames NPC.

The bytes you want to replace for this are going to be depending on which Sprite Property you want to have:

| Byte Sequence  | Corresponding Property | Example |
| ------------- | ------------- |  ------------- |
| `00 00 00 00 00 00 00 00 C0 B2 1F 02` | 32x32 Sprite with 16 frames | Regular NPCs |
| `05 00 00 00 00 00 00 00 C0 B2 1F 02` | 64x64 Sprite with 2 frames  | DPPt larger Legendaries like Palkia and Dialga |
| `0B 00 00 00 15 00 00 00 64 B1 1F 02` | 128x64 Sprite with 2 frames  | DPPt Giratina Distortion World sprite |

You can paste them with Ctrl+B in the green section of the row shown above. Then, save the overlay and save the ROM from DSPRE. If you have done the process correctly, it will show up in game.
If the two frames are different you can set the OW movement to *20-Running Up, Down* in DSPRE event editor to have the animation play out constantly.

### HGSS

- Overworld Properties
  - [Effects](#properties-effects)
  - [Graphics behaviour](#properties-graphics-behaviour)
  - [Rendering setup](#properties-rendering-setup)
- [Following Pokémon](#following-pokémon)
- [Adding new Overworlds](#overworld-expansion)

In HGSS there *appears* to be only one table, it's at offset `0x21BA8` / RAM `0x022074A8`; 901 six-byte entries plus terminator. However, there are a number of related tables for the properties definitions. `Overlay 1` needs to be uncompressed for the following offsets to be accurate. From version 2.0 of DSPRE all overlays are decompressed when unpacking the ROM (alternatively you can use **blz** or an older DSPRE version to decompress it manually).  

The table will look like this:

![](resources/hgss_owtable.PNG)

There are 6 bytes for each entry, laid out as below:

| Entry offset | Size | Meaning |
| --- | --- | --- |
| `+0` (red) | 2 bytes | OW appearance ID |
| `+2` (green) | 2 bytes | Graphics member in `a/0/8/1` |
| `+4` (blue) | 2 bytes | Packed properties |

The two-byte **packed properties** value contains three indices: **effects, graphics behaviour and rendering setup**. These define all overworld behaviour (shadows, beach footsteps, reflections; which model to apply the graphics to; number of frames and how the animations use them).

| Properties bits | Index calculation | Selects | Overlay offset / RAM address | Table layout |
| --- | --- | --- | --- | --- |
| **0–4** | `properties & 0x1F` | Effects preset | `0x21400` / `0x02206D00` | 10 × 2 bytes |
| **5–9** | `(properties >> 5) & 0x1F` | Graphics behaviour | `0x24138` / `0x02209A38` | 20 × 4-byte pointers |
| **10–15** | `(properties >> 10) & 0x3F` | Rendering setup | `0x21A18` / `0x02207318` | 24 × 8 bytes |

Example: **`0x4E27`** (bytes `27 4E`) selects **effects 7, behaviour 17, rendering 19**.

#### Properties: Effects

Defined by indices in bits 0-4.  
Chooses **shadow style, footprints/tyre tracks and whether reflections are enabled**.

<details>
<summary>Effects (details)</summary>

| OW bits 4–0 | Index | Table value | OW count | Shadow | Ground prints | Reflections | Example OW IDs |
| --- | ---: | --- | ---: | --- | --- | --- | --- |
| `00000` | 0 | `0x0891` | 221 | Standard | Normal footprints | Enabled | `1` NPC; `84` rock; `86` tree |
| `00001` | 1 | `0x0911` | 4 | Standard | Cycling tracks | Enabled | `38`/`39` cycling NPCs; `21`/`98` cycling players |
| `00010` | 2 | `0x0012` | 1 | Standard | Disabled | Disabled | `183` special appearance |
| `00011` | 3 | `0x0000` | 1 | Disabled | Disabled | Disabled | `8192` camera focus |
| `00100` | 4 | `0x0002` | 25 | Disabled | Disabled | Disabled | `234` object; `350` trophy |
| `00101` | 5 | `0x0001` | 9 | Disabled | Disabled | Disabled | `415` follower placeholder; `425`/`426` legendary objects |
| `00110` | 6 | `0x0801` | 13 | Disabled | Disabled | Enabled | `479` Diglett; `480` Dugtrio |
| `00111` | 7 | `0x0811` | 593 | Standard | Disabled | Enabled | `428` Bulbasaur; `262` Apricorn tree |
| `01000` | 8 | `0x0821` | 34 | Offset | Disabled | Enabled | `843` Kyogre; `782` Wailord |
| `01001` | 9 | `0x0000` | 0 | Disabled | Disabled | Disabled | Unused |
| **Total** | | | **901** | | | | |

Effects depend on terrain and actor state; large Pokémon still use normal-sized footprints ad shadows. Counts are OW *definitions*, not distinct **graphics files** (some OW definitions use the same graphics).  

Each preset index selects another halfword at `0x21400 + index × 2`:

| Selected settings bits | Meaning |
| --- | --- |
| 0–3 | Some kiind of category* |
| 4–6 | Shadow: `0` none, `1` standard, `2` offset (this shadow is slightly further back than the standard one) |
| 7–10 | Ground prints: `0` none, `1` footprints, `2` tyre tracks |
| 11–12 | Reflection permission: `0` disabled, `1` enabled |
| 13–15 | Zero in all vanilla entries |

*Presets with identical visible effects can differ in their category value. 

:::info  
Followers (**map-object ID 253**) use different reflection routines. The same OW appearance on a non-follower may reflect differently or not at all.  
:::

</details>

#### Properties: Graphics behaviour

Defined by indices in bits 5-9.  
Controls **which animation plays and how it plays**, according to facing and animation state. Other game logic chooses when to switch OW appearances.

<details>
<summary>Graphics behaviours (details)</summary>

Each preset index selects a group of five function pointers at `0x24138 + index × 4` that determine behaviour under different conditions/triggers:

Aside from interaction-specific player behaviours (related to fishing, climbing ladders, saving etc.), the following are the most important "behaviours" to be aware of when configuring new NPCs or Pokémon:
- **`0`** ordinary NPC 
- **`1`** fixed-frame object  (no facing/walking changes)
- **`13`** static (large) Ho-Oh/Lugia  
- **`16`/`17`** Pokémon (equivalent behaviour)

| Index | Overlay offset / RAM address | OW count | Vanilla users / examples |
| ---: | --- | ---: | --- |
| 0 | `0x21428` / `0x02206D28` | 190 | NPC: `1`, `38`/`39` |
| 1 | `0x214B4` / `0x02206DB4` | 26 | Objects: `84` rock, `87` item ball, `86` tree |
| 2 | `0x214C8` / `0x02206DC8` | 0 | Unused |
| 3 | `0x21540` / `0x02206E40` | 10 | Player: Main `0`/`97`, `238`/`239` |
| 4 | `0x2157C` / `0x02206E7C` | 2 | Player: Cycling `21`/`98` |
| 5 | `0x21478` / `0x02206D78` | 6 | Player: Saving `176`/`177`, `198`/`199` |
| 6 | `0x214DC` / `0x02206DDC` | 2 | Player: Watering Can (DPPt leftover) `180`/`181` |
| 7 | `0x21554` / `0x02206E54` | 2 | Player: Surf `178`/`179` |
| 8 | `0x21518` / `0x02206E18` | 2 | Player: Fishing `188`/`189` |
| 9 | `0x214A0` / `0x02206DA0` | 2 | Player: Ladder climb `248`/`249` |
| 10 | `0x21568` / `0x02206E68` | 4 | Player: Heal `200`/`201`, `260`/`261` |
| 11 | `0x21464` / `0x02206D64` | 2 | Player: DPPt Leftover `196`/`197` |
| 12 | `0x21450` / `0x02206D50` | 1 | NPC: Pokémon Centre attendant `335` |
| 13 | `0x2143C` / `0x02206D3C` | 2 | Static Ho-Oh/Lugia `425`/`426` |
| 14 | `0x2152C` / `0x02206E2C` | 0 | Unused |
| 15 | `0x21504` / `0x02206E04` | 12 | 3D overworld objects (not "normal" sprites): `183`, `251`-`257`, `288`-`289`, `291`-`292` |
| 16 | `0x21414` / `0x02206D14` | 34 | Follower: Large `843`, `782` |
| 17 | `0x21590` / `0x02206E90` | 594 | Follower: Standard `428`, `479` |
| 18 | `0x214F0` / `0x02206DF0` | 8 | Objects: Apricorn trees `262`–`269` |
| 19 | `0x2148C` / `0x02206D8C` | 2 | Player: Apricorn tree shake `423`/`424` |

</details>

#### Properties: Rendering setup

Defined by indices in bits 10-15.  
Chooses **the model/texture layout and the data defining each animation’s frames**.

<details>
<summary>Rendering setup (details)</summary>

Read the eight-byte entry at `0x21A18 + index × 8`:

| Rendering entry offset | Size | Meaning |
| --- | --- | --- |
| `+0` | 2 bytes | Zero in all 24 vanilla entries |
| `+2` | 1 byte | Model selection ID, resolved through the intermediate model mapping table at `0x21994` / `0x02207294` to a BMD0 member in `a/0/8/1` |
| `+3` | 1 byte | Frame-map selection ID, resolved through the intermediate frame-map mapping table at `0x219CC` / `0x022072CC` to a frame-map member in `a/0/8/1` |
| `+4` | 4 bytes | Pointer to animation sequences |

Both maps end with selection ID `0xFF`; look up the selection ID rather than treating it as a member number.

| Map | Selector → archive member |
| --- | --- |
| Models | 0 → 266; 1 → 267; 2 → 268; 5 → 269; 6 → 270; 11 → 271; 7 → 272; 8 → 273; 9 → 274; 10 → 269; 12 → 276; 13 → 278; 15 → 279 |
| Frame maps | 0 → 280; 1 → 281; 3 → 282; 4 → 283; 5 → 284; 6 → 285; 10 → 286; 23 → 287; 11 → 288; 13 → 289; 14 → 290; 15 → 291; 21 → 292; 22 → 292; 24 → 293; 26 → 294; 27 → 295; 28 → 296 |

| Rendering index | OW count | Model archive member | Frame-map archive member | Sequence overlay offset / RAM address | Example OW |
| ---: | ---: | --- | --- | --- | --- |
| 0 | 198 | 266 | 280 | `0x217D8` / `0x022070D8` | NPC: `1`; Objects: `86` tree |
| 1 | 2 | 267 | 280 | `0x217D8` / `0x022070D8` | Objects: `84` rock, `87` item ball |
| 2 | 1 | 268 | 280 | `0x217D8` / `0x022070D8` | Objects: `85` breakable rock |
| 3 | 1 | 276 | 280 | `0x217D8` / `0x022070D8` | Objects: `250` |
| 4 | 1 | 279 | 280 | `0x217D8` / `0x022070D8` | Objects: `290` door |
| 5 | 1 | 266 | 288 | `0x21814` / `0x02207114` | NPC: Pokémon Centre attendant `335` |
| 6 | 2 | 266 | 282 | `0x218C8` / `0x022071C8` | Player: Cycling `21`/`98` |
| 7 | 10 | 266 | 281 | `0x2185C` / `0x0220715C` | Player: Main `0`/`97`, `238`/`239` |
| 8 | 2 | 266 | 283 | `0x215BC` / `0x02206EBC` | Player: Saving `176`/`177` |
| 9 | 2 | 266 | 285 | `0x2179C` / `0x0220709C` | Player: Surf `178`/`179` |
| 10 | 2 | 266 | 284 | `0x215F8` / `0x02206EF8` | Player: Watering Can (DPPt leftover) `180`/`181` |
| 11 | 2 | 270 | 286 | `0x216AC` / `0x02206FAC` | Player: Fishing `188`/`189` |
| 12 | 2 | 271 | 287 | `0x21724` / `0x02207024` | Player: Ladder climb `248`/`249` |
| 13 | 4 | 266 | 291 | `0x2161C` / `0x02206F1C` | Player: Heal `200`/`201`, `260`/`261` |
| 14 | 2 | 266 | 294 | `0x216E8` / `0x02206FE8` | Player: Apricorn tree shake `423`/`424` |
| 15 | 2 | 266 | 289 | `0x21760` / `0x02207060` | Player: DPPt Leftover `196`/`197` |
| 16 | 4 | 266 | 290 | `0x215A4` / `0x02206EA4` | Player: Saving `198`/`199` |
| 17 | 1 | 269 | 295 | `0x215D4` / `0x02206ED4` | Static Ho-Oh `425` |
| 18 | 1 | 269 | 296 | `0x215D4` / `0x02206ED4` | Static Lugia `426` |
| 19 | 591 | 266 | 292 | `0x21670` / `0x02206F70` | Follower: Standard `428`, `479` |
| 20 | 37 | 269 | 292 | `0x21670` / `0x02206F70` | Follower: Large `843`, `782`; placeholders `418`–`420` |
| 21 | 8 | 278 | 293 | `0x21640` / `0x02206F40` | Objects: Apricorn trees `262`–`269` |
| 22 | 7 | 278 | 280 | `0x217D8` / `0x022070D8` | Objects: `270`–`276` |
| 23 | 5 | 269 | 280 | `0x217D8` / `0x022070D8` | Objects: Snorlax `395`; `349` |

Index `63` means this table **is not used.** Vanilla uses it for graphics behavious `15` (3D overworlds) objects and camera focus (OW 8192), whose graphics are handled separately.

**Animation sequences:** at each listed sequence address, records are three 32-bit words: start, end and end mode (0 loop, 1 hold; mode 2 terminates the list). Setups 19/20 share four looping ranges: 0–19, 20–39, 40–59 and 60–79. Their model is the only descriptor difference.

**Frame maps:** in the listed `a/0/8/1` member, offset `+0` is a 32-bit count `N`, followed by `N` 16-bit timeline thresholds, `N` 8-bit texture indices and `N` 8-bit palette indices. These map animation positions to images; neither the number of mapping entries nor the sequence duration is the number of unique stored frames. Callbacks control playback speed and progression.

</details>

#### Choosing properties for a replacement

Copy properties from a vanilla OW with matching graphics and animation, then adjust the effects index if needed.

Example: replace an unused Bulbasaur with an ordinary NPC

For the original `1_201.BTX0` example, the graphics member is 201 (`C9 00`). Its OW entry has graphics at overlay offset `0x22EC4` and properties at `0x22EC6`. After replacing its assets with a compatible ordinary NPC, replace properties `25 4E` with `00 00`. The equivalent properties offsets for graphics members 202/203 are `0x22ECC` / `0x22ED2`. Save the decompressed overlay and rebuild the ROM.

#### Following Pokémon

In HGSS, certain Overworld IDs are used for displaying the "following Pokémon", this needs to account for cases where there are visual differences between forms or genders. Three separate ARM9 table map **species, form and gender** to follower OW ID. 

<details>
<summary>Follower mapping tables</summary>

The overworld ID of each following Pokémon is calculated as:  

**OW ID = 428 + species offset + form/female adjustment**; Kyogre: 428 + 415 = 843.

For species with a female variant, add 1 for female; otherwise add the form index (above-maximum forms fall back to 0).  

| Table | ARM9 offset / RAM address | Layout / index |
| --- | --- | --- |
| Species offset (`sModelIndexLUT`) | `0xFF088` / `0x020FF088` | 494 × u16, indexed by species number including 0 |
| Maximum form index (`sFormMaxLUT`) | `0xFE8D4` / `0x020FE8D4` | 493 × u16, indexed by species − 1 |
| Female variant flag (`sFemaleFlagLUT`) | `0xFECAE` / `0x020FECAE` | 493 × u16, indexed by species − 1 |

</details>

#### Overworld expansion

The below details how to expand the number of overworld definitions in the game.  

A hack that adds or removes forms or visual gender differences or species, may need to edit and/or expand the following Pokémon tables.  

<details>
<summary>Expanding the overworld definition table</summary>  

To expand the overlay definition table, copy the entire contents (from `0x21BA8` to `0x230CB` inclusive) to a new location with room for growth (e.g. reclaimed ARM9 or synthetic overlay), and insert new entries *before* the terminator (`FF FF 00 00 00 FC` at offset: `0x230C6`). Then change the following two pointers to the new definition table.

| Item | Overlay offset / RAM address |
|---|---|
| Pointer used by `ObjectEvent_GetGraphicsInfo` | **`0x139FC`** / `0x021F92FC` |
| Pointer used by rendering-descriptor lookup | **`0x14980`** / `0x021FA280` |

</details>  
  
<details>
<summary>Expanding the following Pokémon tables</summary>

To relocate these tables, update the following four-byte pointers to their new **RAM addresses** (little-endian):

| Pointer to | ARM9 offset / RAM address | Current pointer bytes |
| --- | --- | --- |
| Species offset table | `0x6A30C` / `0x0206A30C` | `88 F0 0F 02` |
| Maximum form index table | `0x6A334` / `0x0206A334` | `D4 E8 0F 02` |
| Female variant flag table | `0x6A35C` / `0x0206A35C` | `AE EC 0F 02` |

Repointing relocates the tables; it does not remove the species/OW limits below.

:::warning  
`FollowMon_GetSpriteID` falls back to Bulbasaur for species outside 1–493 or a resulting OW above 993. Appending an OW does not update these mappings or that upper-bound check.  
:::

</details>  