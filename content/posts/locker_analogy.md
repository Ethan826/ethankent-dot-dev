---
title: "Locker_analogy"
date: 2020-01-24T13:49:29-06:00
draft: true
---

## An extremely long analogy

Consider this the director's cut of a strange story that includes a variety
of alternative endings. The scene is set in a school with many floors and
hallways, all lined with lockers. On the first day of class, you are trying
to find a locker in which to store your backpack.

You say good morning to the teacher, and ask where you should put your
backpack. The teacher returns your greeting, and tells you something unusual:
"I don't assign lockers. For that, we use a device called the BackpackHasher.
You tell it a bit about your backpack—the brand, color, and it gives you a
locker assignment."

"Shouldn't I just put my backpack in the first empty locker I find," you ask.

"No, there are a lot of students in this class, so finding an empty locker can
take a long time. It's a lot quicker to use the BackpackHasher, because it
will tell you directly where to go."

"What if I forget where I put my backpack, or what if somebody else has to
get it for me," you ask.

"They can just describe your backpack to the BackpackHasher and it will tell
them the same locker number it told you. The locker assignments seem random,
but as long as a backpack is described the same way, the BackpackHasher comes
up with the same locker number."

You wait in line for the teacher's strange machine, then you describe your
backpack, and the BackpackHasher tells you, "go to locker 39."

"Weird," you think, "didn't the BackpackHasher tell that other student to go to
locker 39 a few minutes ago? In fact, I think I heard a _few_ students get locker
39."

### Alternate ending 1 (hashtable with pointers to arrays)

You walk directly to locker 39—it seems too small for a backpack,
actually—and open it. The locker is indeed too small to hold your backpack,
but it contains a note: "Go to the third floor hallway south of Mr. Jones's
room. Put your backpack in the first open locker in that hallway." You see
another student opening his locker, and it contains a similar note, but
his note refers to the sixth floor hallway north of Ms. Smith's classroom.

You're a pretty fast walker, and you get to the third floor pretty quickly.
As you scan the hallway, you can tell that lockers A–E already have backpacks
in them, but locker F is sitting open. You put your backpack there and head
to class.

The schoolday ends, and you realize you forgot where you put your backpack.
But you describe your backpack to the BackpackHasher, which tells you to go
to locker 39, which has the note telling you to go to the third floor hallway
south of Mr. Jones's room.

As you arrive, you see another student take her backpack out of locker C. A
worker in a uniform marked "Facilities" is watching, and quickly opens all
the lockers after C. He moves the backpack in locker H into locker G, the
backpack from locker G into locker F, and so on, until he closes up the gap
caused by the student removing her backpack. He leaves the first empty locker
door open and closes the rest. (You couldn't see the backpacks from where you
were standing when the Facilities worker moved them around.)

You realize you are going to have to find your backpack by starting at the
first locker and looking in each locker in turn: the backpacks aren't
arranged in lockers in any particular order. But this is still easier than
searching the huge number of lockers in the hallway outside homeroom.

"Pardon me, what happens if I go looking for my backpack but it isn't here,"
you ask.

"Well, you get to an empty locker and don't find your backpack, that's how
you know," responds the Facilities person. "This works pretty well, except
that sometimes all the lockers get filled up. Then we have to get all the
backpacks out of the lockers and carry them to a hallway with more lockers,
and then go back and update the instructions in the locker that sent you over
here."

### Alternate ending 2 (hashtable with pointers to linked lists)

Just like in ending 1, you are directed to the third-floor hallway. But this
time, you are explicitly told to open Locker J rather than find the first
empty locker in the hallway. Locker J already has a backpack, but it has a
laminated note hanging from a ring: "If I'm full, go to the second floor,
Locker N, and put your backpack there." (The floor and locker number are
handwritten onto the laminated card. There are some additional instructions,
but they relate to retrieving your backpack rather than putting it away.
You'll check those out later.)

You walk to Locker N on the second floor, which directs you to Locker Z on
basement level 1. But Locker Z's instructions don't tell you where to go
next. You see a person in a "Facilities" uniform. "Pardon me. This locker is
full, but the instructions don't tell me where to go next."

"Go, to, umm, how about Locker V in the cafeteria? Yeah, that one." You can
see he is looking at a list on his clipboard marked "Available Lockers." He
crosses out "Locker V" on his list, and updates the instruction card in
Locker Z so that it refers to Locker V throughout. Locker V is empty, and you
put away your backpack

At the end of the schoolday, you've forgotten your locker's location, but you
describe your backpack to the BackpackHasher, which sends you to Locker 39,
which points you to Locker J, then Locker N, then Locker Z. Locker Z's
instructions tell you to go to Locker V.

As you're about to leave for Locker V, you notice the rest of the
instructions on the laminated card: "Take me with you." You realize the ring
attached to the instruction card is on a cord, and the cord retracts
automatically (like a seatbelt). You pull it out a little bit and it zips
back.

So you take the Locker Z instruction card with you to Locker V. It's a long
cord! You open Locker Z and discover your backpack. Before you close the
locker, you read the rest of the instructions on the card you took with you:
"Call a Facilities worker for help."

You see a Facilities worker and ask for help. She takes the instruction card
tethered to Locker Z. She erases all references to Locker V on the
instruction card. She looks at the instructions in your locker, Locker V, and
sees that they refer to Locker Q (which you've never seen before).

On the instruction card you handed her—the one for Locker Z—she writes "Q"
everywhere "V" used to be. Now it has exactly the same instructions as the
card for your locker, Locker V. She lets go of the instructions on the
lanyard, and it weaves its way through the school, making a satisfying "zip"
sound as it goes. Finally, the Facilities worker erases the instruction card
inside your locker, Locker V, then adds "Locker V" back to the list of "Empty
Lockers." In short, the chain of instructions now skip your locker, and it is
back in the pool of empty lockers.

"What was that all about," you ask.

"Well, whoever came after you this morning saw your bag hanging in this
locker, Locker V, and we found that student a new locker, just like we did for
you. That student got Locker Q, as it happens. But now your old locker," she
points to Locker V, "isn't in use. So there's no need for the student who
came after you to bother stopping here. That's why we rigged up the
retractable cords. Now we can just rewrite the instructions about where to go
rather than actually moving any bags around. And we can make the chain of
instructions however long it needs to be because we aren't restricted to
lockers next to each other that fit in a hallway. But it does make you
students have to do a lot of walking around sometimes."

### Alternate ending 3 (hashtable with linear probing and tombstones)

As before, the BackpackHasher tells you to go to Locker 39 right next your
homeroom classroom. This time, it's large enough to hold a backpack—but
it's already full. There's a note: "Put your backpack in the first empty
higher-numbered locker." (You ignore some instructions about retrieving your
backpack.) Locker 40 and 41 are full, but Locker 42 is empty.

At the end of the schoolday, you describe your backpack to the BackpackHasher
and are told to go to Locker 39. The retrieval instructions say, "If this
isn't your backpack, check the next higher-numbered locker. If you reach an
empty locker, your backpack isn't here."

You find yourA
