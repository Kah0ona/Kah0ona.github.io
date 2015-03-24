---
layout: post
title:  "A journey into Haskell: Harmony, a library for music"
date:   2015-02-23 19:19:02
categories: software Haskell
---

Recently I felt the urge again to work on a little pet project. I decided to give [Haskell](https://www.haskell.org/) a go,
and I loved it. I read some articles and books about it, followed [learnyouahaskell.com](http://learnyouahaskell.com) and 
played around in the REPL, but nothing speeds up the learning process more than making something useful.

Another hobby of mine is playing guitar. I'm not actually very good at it, nor do I know a lot about musical theory, 
but I can crank out the odd Bob Dylan song. 
 

Building blocks of music: notes, chords and scales
==================================================

**TL;DR; (Western) music consists of 12 possible notes. Combinations of notes form chords and scales.**

I decided I wanted to create a type-safe library that can generate and manipulate musical scales and chords. Therefore, first some very basic 
musical theory. Musical theory can be very complex, but luckily I'm a bit of a novice myself so I can't get carried away. 
I try to give just enough information needed to describe the intent of my project.

In music there are 12 different **notes**: 
A, A#, B, C, C#, D, D#, E, F, F#, G, G#

See the ones with the #? They are called 'sharps', and on a piano, those are the black keys. The 'normal' notes correspond to the white keys. 
They repeat, so the succesor note of G# is A again. If you think you are missing both a B# and a E#, you are right, but there is some [complicated theory](http://en.wikipedia.org/wiki/Scale_(music)) behind that of why those notes do not exist.  

Then there are two more words to explain, before we get to the code: **chords** and **scales**. 

A chord is played when you press three or more keys at the same time on a piano, or, if you play on a guitar, strum multiple strings at the same time. 

A scale is the thing you know from school, do re mi fa sol la ti do. A scale and a chord are both subsets of the 12 possible notes. 
Generally a chord comprises of around 3-5 notes, where as a scale is more likely to have around 8 notes. 
The notes in a scale aren't meant to be played at once, but are useful as building blocks (or rather 'ingredients') for melodies.

Together, chords and melodies make songs.


Modeling music in Haskell
=========================

I wanted to get my feet wet on Algebraic Data Types. So I started out by modeling the notes. 

	data Note = A | A' | B | C | C' | D | D' | E | F | F' | G | G' deriving (Show, Ord, Eq, Enum)

`#`s are not allowed, so I used apostrofe (') for the sharp sign. Notes derive Enum, so we can define ranges as well:

	*Harmony> [A .. E] 
	[A,A',B,C,C',D,D',E]

Pretty cool. Alright, next up, we need to define a scale. Basically a scale is a list of notes, 
but to me it would be nice if we can label it properly, so I used a newtype:

	newtype Scale a = Scale [a] deriving (Show,Ord,Eq)

**Note:** I couldn't figure out how to restrict this more such that only Notes can be put in a scale, 
but I worked around this by putting type signatures on the functions that work on scales, as I show below.

Then, to be able to `fmap` over a Scale, we make sure it implements the `Functor` typeclass:
 
	instance Functor Scale where
		fmap f (Scale []) = Scale []
		fmap f (Scale (x:xs)) = (Scale ((f x):(fmap f xs)))

With this, we have the types ligned up nicely, so let's continue to to make some functions that does interesting stuff. Remember that I said that notes are circular, ie. after G#, comes A again. Let's implement this:
	
	-- Gets next chromatic note
	next :: Note -> Note
	next G' = A
	next x  = succ x

	-- Gets previous chromatic note
	prev :: Note -> Note
	prev A = G'
	prev x = pred x

The word chromatic means 'smallest step'. On a piano this means the directly adjacent key to the right for `next` and left for `prev`. Let's test it: 

	*Harmony> next A
	A'
	*Harmony> next G'
	A
	*Harmony> prev B
	A'
	*Harmony> prev A
	G'

A useful helper function will be a function that gets the note `n` chromatic steps away:

	noteAtInterval :: Note -> Int -> Note
	noteAtInterval baseNote n = iterate next baseNote !! n

Eg.

	*Harmony> noteAtInterval C 3
	D'

Alright, lets create a Scale! I've written the following function that does exactly that

	createScale :: Note -> [Int] -> Scale Note
	createScale root list  = (Scale (map (noteAtInterval root) list))

Woah, why not supply a few notes, why work with a list of `Int`s? Because scales generally have root note, which is the first note. 
The most well known scale (do re mi...) is called a major scale. It comprises of 8 notes, starting from a specified root note. 
For C Major, this will be the notes [C,D,E,F,G,A,B,C], so to get those, starting to count from C, we would need the notes at the indices 
[0,2,4,5,7,9,11,12]. In fact, starting from any note X as root, those indices yield the corresponding major scale of that note. Thus, we can
define a majorScale as follows:

	majorScale :: Note -> Scale Note
	majorScale root = createScale root [0,2,4,5,7,9,11,12]

Let's test: 

	*Harmony> majorScale C
	Scale [C,D,E,F,G,A,B,C]	

Woop Woop! Another, E Major scale:

	*Harmony> majorScale E
	Scale [E,F',G',A,B,C',D',E]

I like. Following this pattern, we can define other scales and chords as well. There are formulae for generating scales and chords based 
on such indices. I've implemented some of the common chords and scales:

	majorScale :: Note -> Scale Note
	majorScale root = createScale root [0,2,4,5,7,9,11,12]

	minorScale :: Note -> Scale Note
	minorScale root = createScale root [0,2,3,5,7,8,10,12]

	majorTriad :: Note -> Scale Note
	majorTriad root = createScale root [0,4,7] 

	minorTriad :: Note -> Scale Note
	minorTriad root = createScale root [0,3,7]

	addedFourth :: Note -> Scale Note
	addedFourth root = createScale root [0,4,5,7]

	sixth :: Note -> Scale Note
	sixth root = createScale root [0,4,7,9]

	sixNine :: Note -> Scale Note
	sixNine root = createScale root [0,4,7,9,2]

	majorSeventh :: Note -> Scale Note
	majorSeventh root = createScale root [0,4,7,10]

	cmaj7 :: Scale Note
	cmaj7 = majorSeventh C 

It is not even close to a complete list, but you get the point. 

Okay, on this [github repo](http://github.com/Kah0ona/harmony-haskell "Harmony github repo"), you will find some more utility functions like `sizeOf`, `scaleContains`, `noteAt`, and `transpose`.
The last one is quite handy, it can transpose a scale from a root X to a root Y.

Since I am a guitar player, I decided I wanted to put some guitar specific functionality in there. To start off,
I would like to model a guitarString. A guitar has frets, those little boxes on the neck. Following one string, from one fret to 
the other in sequence, the tone keeps increasing chromatically. A guitar has around 20 frets, so it can go 
round the 12 note spectrum almost twice per string. Multiply this by 6 strings on one guitar, and there are like 6x20 notes available for play.

Modeling a guitar string is easy, it's just a chromatic scale of 20 steps, starting from some root note N (this is the note that you hear if you play the string without pressing a fret). Thus:

	 
	guitarString :: Note -> Scale Note
	guitarString root = createScale root $ [0..20]

If we tune a string to E, we'd use:
	
	*Harmony> guitarString E
	Scale [E,F,F',G,G',A,A',B,C,C',D,D',E,E,F,F',G,G',A,A',B,C,C',D,D',E]

So what about a complete guitar then? A normal guitar, with standard tuning, has 6 strings. From thick to thin they are tuned E, A, D, G, B, E:
	
	standardTuning :: [Scale Note]
	standardTuning = map guitarString [E,A,D,G,B,E]

Test 1,2,3...

	[
	 Scale [E,F,F',G,G',A,A',B,C,C',D,D',E,E,F,F',G,G',A,A',B,C,C',D,D',E],
	 Scale [A,A',B,C,C',D,D',E,F,F',G,G',A,A,A',B,C,C',D,D',E,F,F',G,G',A],
	 Scale [D,D',E,F,F',G,G',A,A',B,C,C',D,D,D',E,F,F',G,G',A,A',B,C,C',D],
	 Scale [G,G',A,A',B,C,C',D,D',E,F,F',G,G,G',A,A',B,C,C',D,D',E,F,F',G],
	 Scale [B,C,C',D,D',E,F,F',G,G',A,A',B,B,C,C',D,D',E,F,F',G,G',A,A',B],
	 Scale [E,F,F',G,G',A,A',B,C,C',D,D',E,E,F,F',G,G',A,A',B,C,C',D,D',E]
	]

Hooray. Okay, so what if we want to play the notes of an E Major scale on the thickest string (which is tuned in E), 
which frets would we have to push? The next function will calculate this:

	getPositionOfNotes :: (Scale Note) -> (Scale Note) -> [Int]
	getPositionOfNotes (Scale []) _ = []
	getPositionOfNotes scale (Scale []) = take (sizeOf scale) $ [0..]
	getPositionOfNotes scale chord = [ x | x <- [0..(sizeOf scale)], 
												(x < (sizeOf scale)) && -- x < number of notes in scale, and...
												 scaleContains chord (noteAt scale x) ] -- the chord 

Let's try:

	*Harmony> getPositionOfNotes (guitarString E) (majorScale E)
	[0,2,4,5,7,9,11,12,13,15,17,18,20,22,24,25]

If you have a guitar, try it out, and sing along the famous `do re mi`. (`0` means strum the string without pushing any fret).

Okay, how about chords. What if we want to play an E major chord on a guitar that has standard tuning? 
The following function gives me that answer. It returns a list of fret numbers to push on each string (thickest string on the left):

	frets :: (Scale Note) -> [Scale Note] -> Int -> [Int]
	frets (Scale []) strings minimumFret = listofzeroes (minimumFret + (length strings)) 
	frets _ [] _ = []
	frets chord strings minimumFret = map (\string -> head $ 
													  filter (>= minimumFret) 
														(getPositionOfNotes string chord)) strings

Let's see: 

	*Harmony> frets (majorTriad E) standardTuning 0
	[0,2,2,1,0,0]

Try this on a guitar in standard tuning, and strum all strings at once. You'll notice it sounds good. Combine it with `majorTriad D` 
and `majorTriad A`, and you'll sound like you know what you're doing. Wondering what the last parameter is?  
Well you see, all chords can be played on multiple places on the neck. If I put it to 0, it'll show the E 
chord most towards the top of the guitar. If I put it to 7, you'll notice a different position of the E Major chord.

	*Harmony> frets (majorTriad E) standardTuning 7
	[7,7,9,9,9,7]

This particular chord is called a barre chord. A guitarist needs to 'bar' his first finger on all 6 strings to be able to play it. 
The advantage is that if you keep your hand in the same 'grip' and move it up the neck, you get the major chords of all the possible notes. 
Plus on an electric guitar you'll sound like a true rock star. 

A common way to communicate how a chord is played on a guitar, is using a following ASCII-diagram:

	0 2 2 1 0 0
	===========
	| | | o | |
	| o o | | |
	| | | | | |
	| | | | | |
	| | | | | |
	| | | | | |	

With some imagination you'll see that it's supposed to be the neck of a guitar, first fret is the first line below the ========. 
The vertical bars are strings, thickest most left. An o means 'press that fret'. 
So the major E Chord above can be played with 3 fingers, leaving the other strings open, and strumming all 6 strings. 
For convenience, a list of fret numbers is also displayed in the first line.

I've implemented a function for generating these diagrams as follows. (I omitted some helper functions, see github for the complete source):

	fretdiagram :: (Scale Note) -> [Scale Note] -> Int -> [Char]
	fretdiagram chord tuning minimumFret =
					let 
						indices = frets chord tuning minimumFret 
						divider = "===========\n"
						header = "\n" ++ (join " " (map show indices)) ++ "\n"
						grid = getDiagramGrid [] indices (minimumFret + (length indices))
						fretrows = map fretrow grid
					in
						header ++ divider ++ (join "\n" fretrows) ++ "\n\n"

Testing it out with the following one-liner, and it'll print the diagram above:

	*Harmony> putStr $ fretdiagram (majorTriad E) standardTuning 0	

That's it. I had a lot of fun writing this small library. The library can be improved by adding more scales, tunings and chords. 
And real musicians could probably dream up way more stuff to put in than I could.


Conclusion
==========
Algebraic Data Types are a great to model the domain. Once I defined a few basic functions around those data types, 
I found it really remarkable how easy more complex functions were implemented. Simply by composing smaller functions.

As a final note, I just love how clean, compact and elegant Haskell code is. Even when a beginner like me writes it. I am pretty sure there is enough space for improvements. But whatever, in around 150 lines of code and a lazy Sunday, 
I created a nifty little toolkit. I program mostly in Java in my day job, and this feels just so much cleaner and efficient.

If you have any comments, tips or hints on this article; or on the code, please let me know.
