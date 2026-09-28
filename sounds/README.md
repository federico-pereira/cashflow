# Sound effects

The game makes all its sounds itself (Web Audio API), so this folder can stay empty.

To replace a generated sound with a real recording:

1. Add the file here as `<name>.mp3`, for example `dice.mp3`. Short files (under 2 seconds) work best.
2. Add the name to `index.json`, for example `["dice", "win"]`.
3. Commit and push. Players need a hard refresh (Ctrl+Shift+R).

Names: `dice`, `payday`, `cost`, `buy`, `sell`, `loan`, `denied`, `deal`, `event`, `charity`, `paid`, `turn`, `win`, `firesale`, `bankrupt`.

| Name | When it plays |
|---|---|
| dice | Anyone rolls (plays with the dice animation beside the board) |
| payday | You collect a positive payday |
| cost | You pay a bill you can't refuse (doodad, Downsized, lawsuit, negative payday) |
| buy / sell | You buy or sell stocks or property |
| loan | You take a bank loan |
| denied | The bank refuses a loan |
| deal | A flash deal pops up for you |
| event | An event hits on your turn |
| charity | You donate |
| paid | You pay off debt (also the unmute test sound) |
| turn | It becomes your turn (online only) |
| win / firesale / bankrupt | Announcements everyone hears |
