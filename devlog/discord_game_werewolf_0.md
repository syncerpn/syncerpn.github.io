---
layout: post
title: werewolf dev log 0
---
I was lazy and out of idea when it comes to making new videos for my youtube channel.
So, I decided to create a version of werewolf board game for my discord server.
The game is text-based in the upcoming first release.
Here is the game rule and features:

## Rule
There are currently 4 roles: Villager, Seer, Guard, and Werewolf.
Werewolves are against the villagers which include Villagers, Seers, and Guards.
Two sides will have to enliminate the each other.
Werewolves are hidden among the villagers.
They also know their partners, but the villagers do not.

The game may starts if there are 7 players: 2 werewolves, 3 villagers, 1 seer, and 1 guard.
The in-game time will alternate between day and night.

### First day
Everybody greets each other.
There is no event in the first day.

### First night
The Villagers will go to sleep at night.

The Seer can check the role of 1 person that she wants.
<pre class="memory">
<span style="color:aqua">!check</span> [id]
</pre>

The Guard can protect 1 person that he wants.
<pre class="memory">
<span style="color:aqua">!protect</span> [id]
</pre>

The Werewolves will discuss and vote to kill 1 person.
<pre class="memory">
<span style="color:aqua">!kill</span> [id]
</pre>

You can also cancel your move.
<pre class="memory">
<span style="color:aqua">!cancel</span>
</pre>

### After everyone makes their move, the change will be applied before the first night ends.
The Seer will know the role of the person she checked.

The person who is chosen by the Werewolves will be killed, unless protected by the Guard.

No matter the protection by the Guard is triggered, this buff wears off next round.

### Next day
If there is anyone killed, it will be announced publicly.
However, the role of that person is non-disclosed.

Everybody should exchange the information they have.
Be aware that the Werewolves will probably lie.

1 person may be hang if there is a majority of votes against the guy.
Or everybody skips this turn.

<pre class="memory">
<span style="color:aqua">!vote</span> [id]
</pre>

### Next night, next next day, and more
Similar to the first night, everyone makes their move and gets the information.

### Game ending
The game will end when there is no alive werewolves. In this case, the villagers win.
The game will also end when the number of werewolves is equal or higher than the number of villagers. In that case, the werewolves win.
