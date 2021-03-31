# HoN Ranked Quickstats Tooltips Mod

![Example animated gif of mousing over the stats button and seeing the floating panel](https://i.imgur.com/Sp0zOO3.gif)

## About

This is a mod for [Heroes of Newerth](http://www.heroesofnewerth.com) to revive a similar feature that the old style UI had: a stats button with a hoverover quickstats tooltip.

## Installing
0. You will need the **latest** [HoN Mod Manager](https://github.com/mrhappyasthma/Heroes-Of-Newerth-Mod-Manager/releases/tag/1.4.0.1) to use this mod. So first, install that.

1. Once that is installed, you can download the latest release of the Ranked Quickstats Tooltip mod from [here](https://github.com/mrhappyasthma/HoN-RankedQuickstats-Tooltips/releases/download/Latest/ranked_quickstats_tooltips.honmod).

2. Place the downloaded `ranked_quickstats_tooltips.honmod` file into your Heroes of Newerth directory under `game/mod` folders. For example: `C:\Program Files (x86)\Heroes of Newerth\game\mods` or `C:\Program Files\Heroes of Newerth x64\game\mods`.

3. Open up `HoN Mod Manager` and enable `Ranked Quickstats Tooltips` mod: <br/><br/>
![Open HoN Mod Manager and click 'enable' for the 'Ranked Quickstats Tooltips' mod.](https://i.imgur.com/xiYSHO3.png) 

4. Click `File -> Apply Mods`: <br/><br/>
![Click the 'File' tab and click on the 'Apply Mods' option.](https://i.imgur.com/N7TweIL.png) <br/><br/>
*Note: You will need to re-apply mods every time HoN is updated.* <br/><br/>
*This is done manually by opening HoN Mod Manager. It will detect the version change and give you a pop-up asking if you'd like to re-apply. Click yes, and you're good to go! If not, you're client WILL be very buggy.*

## Updating
Updating is also done through the `HoN Mod Manager`. Simply click on `File -> Download Mod Updates`: <br/><br/>
![Click the 'File' tab and click on the 'Download Mod Updates' option.](http://i.imgur.com/rbdQZzu.png)

## Known Issues

The list of known issues is plentiful.

1. This mod only works for the new interface UI. This is by design, but could potentially be ported to the legacy UI.
2. The 'Match History' section is empty. `PlayerStatsNormalSeasonResult` provides recent match IDs (`param[52]`), but not all the individual fields expected by the UI. It expects `7x` of the following `[path_to_hero_icon, ^844Win/^522Lose, kills_in_match, deaths_in_match, assists_in_match]`. I need to find a way to fetch this data from the match ID using Lua or a trigger.
3. The positioning/sizing of the panels is hacky as can be. I couldn't find the most appropraite place to put this panel in the hierarchy, so I primarily just changed the `x`, `y`, `width`, `height` in order to make something that looks reasonable. This can easily be improved.
4. Some icons are missing (e.g. `electrician`). The icon seems to be stored at a different path `.../icons/hero.tga` instead of `.../icon.tga`. And it's not clear why or if there are others. Most* seem to work.
5. This always shows the stats from the current season, even if the user is in a Midwars, Public, or Casual match. This could be fixed but since I only play ranked games primarily, I cut corners by desining it to exclude game variant.
6. The 'New Player' overlay is not ported. The code can mostly be forked from `game_lobby.package`, but I had some trouble getting it to work without bugs so I just skipped that for now.
7. The GPM field in the UI is misaligned (as seen in the screenshots). This is a preexisting issue that affected the UI element in `game_lobby.package` that I forked from. But it may be nice to fix this.

## Design

This mod is not particularly well written. I had to reverse engineer a way to trigger the data to be sent in a format that could be mapped to the existing UI elements that I forked form the old style UI (`game_lobby.package`).

I also rely VERY heavily on the use of `Cvars` to store/pass parameters. Which is not ideal, but I couldn't find a better way.

Essentially it works like this:

- First, I created a new Lua function in `player_stats_v2.lua` which calls some internal functions to trigger a stats fetch. This will eventually issue the `trigger` for `PlayerStatusNormalSeasonResult` (`Player_Stats_V2:FetchSeasonNormalData(<username>)`.). This is what we will `watch` in UI code.

- Second, I ported over a few UI elements (with minor tweaks) from `game_lobby.package` including the `stats_button` and ``.

- Third, I wrote most of the logic on the `stats_button` itself. This works sort of like a state machine.

    - A.) Whenever entering a lobby, watch for `LobbyPlayerInfo{index}` trigger. This is done when the game starts the picking phase and is triggered once for each user (corresponding to `index`.

    - B.) Whenever that triggers, store each player's name and color in a `Cvar`. (`QuickstatsMod_stats_tipPlayerName{index}` and `QuickstatsMod_stats_tipPlayerColor{index}`).

    - C.) Whenever the user mouses over the stats icon trigger `DoEvent(0)`.

    - D.) `onevent0lua` we call in to the custom Lua function that I wrote ` Player_Stats_V2:FetchSeasonNormalData(<username>)`.

    - E.) This Lua code issues the actual stats search by faking a form submissions. This eventually causes the game engine to trigger `PlayerStatusNormalSeasonResult` with all the userful parameters that we need.

    - F.) The `stats_button` second watch is on `watch2="PlayerStatsNormalSeasonResult"` to catch and handle these triggers. Since every stats button will receive every trigger notification, we have to ensure only the one specific to the hovered element is passed to the floating dialog. This is done using helper `Cvar`s that track which element is moused over.

    - G.) In `ontrigger2`, in response to `PlayerStatsNormalSeasonResult`, we parse all of the parameters (which can be reverse engineered by looking in the Lua files) to match the expected format that the forked code from `game_lobby.package` expects. (Username, KDA, GPM, XPM, etc.) When finished, this trigger posts `DoEvent(1)`.

    - F.) The final state `DoEvent(1)` (i.e. `onevent=`) issues a few triggeres for each UI state that needs updated on the floating panel. This mimicks what `game_lobby.package` did. There's a separate trigger for each portion of the UI.

    - H.) Lastly, the `QuickstatsMod_stats_floating_panel` (which is largely forked from `game_lobby.package`) responds to these 'data is ready' triggers and updates the text in the appropriate UI elements and fades them in.
