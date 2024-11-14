---
layout: post
title:  "[code] Bépo Steno: the day I tried to type at the speed of speech"
date:   2024-11-11 09:09:36 -0400
categories: Plover steno Grandjean
---

# Bépo Steno: a french stenography plugin

Three years ago, I put together [this project](https://github.com/AntoineBalaine/bepo-steno?tab=readme-ov-file).
It’s a plugin for a stenography software called plover.
My world-renowned impatience had pushed me towards figuring out a way to type faster (and more accurately) so I decided I wanted to give steno try.

## TLDR:
If you’re considering learning steno for every day use, it’s not worth it - unless you’re planning on becoming a court reporter.
There’s too many roadblocks on a computer - your apps are not ready for this type of transition, plus it takes a long time to become proficient with steno. A _very_ long time, which promises to be marked by the constant frustration of fighting the apps that are unfit for steno inputs.

So dear reader, consider this piece a curiosity, a fun fact to throw in conversation at a party - nothing more.

# Why would you do that?
The premise is: why type one letter at a time, when you could be hitting multiple keys simultaneously?
This is an age-old issue because, in order to do that, you need a keyboard layout that actually allows it: you need a layout that can hit the letters in the same order as they appear in words. For the English language, it’s not really possible without a dedicated steno keyboard (not even with Colemak or Dvorak layouts), but in french it is actually possible to do with a regular one.

# How does it work?
As I was looking into steno, I wondered if it was possible to do it on a regular keyboard - that is, type on a regular keyboard at the speed of speech, by hitting multiple keys at once.

There’s this non-standard french layout called [Bépo](https://bepo.fr/wiki/Accueil). It’s built around a frequency analysis of the letters: the most used letters are in the center row, the vowels are in the left hand, and the consonants are in the right hand.

I found that if I tweaked the layout a bit, it was possible to use this 100-year old french steno technique, called the Grandjean method. 

en
# Credits, acknowledgements
The method was demonstrated at length by an abandoned [github repository](https://github.com/azizyemloul/plover-france-dict#%C3%A0-propos-de-la-st%C3%A9notypie). The repo had an initial implementation of the Grandjean Method for Plover, it had been written by a student of the steno school in France, it had been thoroughly documented, and abandoned most likely when the student dropped out.
It’s been wonderful to find this repo, because it provided detailed information about the technique which would have otherwise been impossible to find: Steno schools and court reporters tend to be very secretive about their trade, which makes it difficult to get info online.

Another invaluable source of information was the author of the youtube channel [StenoTypist](), who granted me a few informational interviews before revealing that he worked for the Steno school in France! I am very grateful for the help and encouragement he provided.


# So what’s this plugin thing?

The plugin relies on [Plover](https://www.openstenoproject.org/plover/), which is the open source project for stenography. Lore has it that the project was endeavoured by an american court reporter who was tired of the software vendor lock-in. Fast forward ten years and some extensive collaboration with MIT students: Plover’s got a large eco-system of plugins, with offerings such as: steno on piano keyboards, multiple languages, steno-coding extensions - all sorts of wonderful things.

For my plugin, I built a very large JSON dictionary (300k+ entries) that matches key strokes to syllables and words, using the rules of the Grandjean method. If you use an ortholinear keyboard (works on a 40% keyboard, such as the Plank keyboard), and do writing in french, it can reach speeds of 170 words per minute.

## What does the method entail?
The point is that Grandjean is a syllabic method. At its core, one multi-key-stroke = one syllable. Each word is represented in a JSON dictionary (key/value pairs) where a series of slash-separated strokes will serve as keys to a word. As you type, the program converts your strokes into words: 
```JSON
TEl/MI/NE: terminé
```

This syllabic technique is NOT the same as the Ward-Ireland technique, which is used in English. That one is word-based: one stroke = one word. Obviously, when comparing the two, there’s exceptions to the rule: french stenos will write custom dictionaries where long words are represented as single strokes, and inversely English-speaking stenos can sometimes break out some words syllabically if they need to. 

## Syllabic versus word-based, which to prefer?
Word-based can achieve faster typing speeds, but it takes much longer to master.
Syllable-based is easier to pick up, but it’s harder to reach the speed of speech with it.

Professional steno is a long and risky endeavour - the drop out rate in steno schools is astoundingly high, on both sides of the channel. Lots of students find themselves hitting a plateau half-way through the curriculum, only to decide the career is not for them. 


## Who’s this plugin for?
Obviously, despite advertising it to professional stenographers and all kinds of writer-nerds, my plugin never caught on. Steno itself is fun to use, but also comes with a ton of constraints. 
For the french pros: they need a more robust solution than this - at least a software that’s more featureful than what I can offer here.
For the small pool of potential aficionados: you need to know french, own an ortholinear keyboard, use the bépo layout, and be into stenography - that’s probably about two guys in the world (including me).  If I were a writer or an academic in the french language, this would probably be a godsend, though.

At least I learned regex when building this. Modifying the example dictionary to fit my modernized method took opening the JSON files in vim, and exerting changes over the key/value pairs in swaths of thousands at a time, using regular expressions.

## Some of the changes I did
The bepo steno layout looks like this:
```
_________________________________________
|  B|  É|  P|  O|  È|  ^|  V|  D|  L|  J|
|___|___|___|___|_ai|___|__*|___|___|__$|___
|  A| U | I | E |  ,| C | T | S | R | N |  M|
|___|___|___|___|___|___|___|___|___|___|___|
|  À|  Y|  X|  .|  K|  '|  Q|  G|  H|  F|
|___|___|__y|___|___|__n|__£|__h|__h|___|
```
And if you were wondering - yes you _do_ need a squared keyboard to type with it. The dedicated term for this type of keyboard is called «ortholinear».

## Adaptations from the original Grandjean method (and plover)

The original Grandjean uses these keys:
`S K P M T F * R N L Y O E A U I l n $ D C`

My plugin uses these keys:

`S K P M T F * R N L Y H O ^ E È A À U I l É n $ B D C <space> . ,`

So there’s a fair amount more there. The original layout allows typing quickly, but this makes transcribing automatically challenging: the key strokes are very phonetic, so for example if you have a set of keystrokes which phonetically sounds as something that could be orthographed multiple ways, you find yourself in a pickle: now your software has to be able to choose which transcription is more likely to be correct. 
These phonetic ambiguities are called homophones (aka «sounds the same»).

As a fix, I included an extra layer of keys which allow dis-ambiguating the homophones. Essentially, they’re keys that tell the software how to spell/conjugate. It wasn’t too difficult to include them (aside from the endless regexes in vim), I just had to attach them to those of the keys that were un-used by the original Grandjean method.

### Step one: reducing dual-use Letters

The Grandjean steno method contains several dual-use ending letters:
- "$" is used to indicate word endings in "s", "z", "f", "v".
- "n" is used for endings in "n" and "m".
- "l" is used for endings in "l" and "r".
- "d" is used for endings in "d", "t", "p", "b".
- "c" is used for endings in "c" and for silent "e".

This causes many ambiguities in the interpretation of steno strokes. To solve this problem and minimize the double and quadruple uses of certain letters, Bépo-steno uses letters which aren’t part of the original Grandjean design.

### Steno Order of Bépo-Steno

Bépo-steno introduces several new letters compared to the Grandjean method to address several daily usage needs.
The modifications are as follows:

#### É and È Keys: Disambiguating Homophones
Because French is an orthographic language before being phonetic, and steno is a phonetic method before being orthographic, any stenographic transcription is bound to suffer from the phonetic ambiguities of the language. That is, the same sound can correspond to several words: "ces", "ses", "c’est", "s’est", "sais", "sait"; and this sound can only be stenographed by a single stroke: "SE". This specificity of French challenges any phonetic transcription method and requires a means to disambiguate the orthographies of the same sound.

  This project uses an "orthography" key, the "É". Placed in the last syllable of a word, it converts the end of the syllable into an orthographic ending. Several rules apply:
  1. if the word has no final consonant, the word is made plural by default. `TEl/MI/NÉE` gives "terminés".
  2. if the word has a final consonant, the consonant is silent. `MÉED` gives "met".
  3. if the final consonant of the word is "$", then the ending becomes "-ez". `TEl/MI/NÉE\$` gives "terminez". NB: this rule is unfortunately not implemented very rigorously in the dictionary, there may be errors when you use this rule.
  4. if the final consonant is a "c", either the steno stroke is translated into a word ending in "C" and made plural: `STUÉc` will give "stucs", or it is translated into a feminine plural word: `MUÉC` will be translated as "mues".

The "É" key is not sufficient to overcome all cases of ambiguities, particularly for verb conjugation in the past tense. "terminai" is stroked the same as "terminé". The ambiguity between the ending in *"ai"* and the ending in *"É"* is present in all past tense conjugations in French and therefore requires a special solution: the "È" key.

The `È` key indicates the "é" sound spelled "ai". It is usable for verbs as well as common nouns, with a difference in usage between the two:
1. for common nouns, "É" follows the normal usage: `LÈ` = "lait", `LÈÉD` = "laid", `LÈD` = "laide", etc.
2. for verbs, `«È»` considers that all final consonants of the stroke are silent - the orthography key does not need to be used. `MAn/YÈ$` = "mangeais", `MAn/YÈD` = "mangeait", `MAn/YÈn` = "mangeaient", etc. Only the ending in "-ez" requires the use of the orthography key: `MAn/YÈÉ$` = "mangez".

#### H Key: Apostrophes and Homophones
- The "H" key has two uses:
1. Placed between the initial consonants and the vowels of the middle of the syllable, it serves to indicate apostrophes: when you strike an initial consonant, the H, and a vowel in the same syllable, Bépo-steno will divide the syllable at the location of the "H" stroke. For example: `LHAn` will be transcribed as "l'an", `KHOn` will be transcribed as "qu'on". Plover will divide the two words into two strokes. **It is imperative to note** that to benefit from this functionality, you will need to install the [plover-split-at-apostrophe](https://github.com/AntoineBalaine/plover-split-at-apostrophe) plugin. To do this, follow the same steps as for the installation of Bépo-steno, but substituting the address with that of plover-split-at-apostrophe:

```bash
plover -s plover_plugins install git+https://github.com/AntoineBalaine/plover-split-at-apostrophe
```
> NB: cycling through homophones was never a goal of the plover project, I remember reading here and there some resistance from the plover devs to include it as a feature. I built my small plugin to do that, opened a PR to have it included in Plover’s plugins list, and am still waiting for feedback some years later.

2. When struck alone, the "H" key allows you to substitute the previous word with another homophone (same sound) from the dictionary. Thus, striking `KA/NAl` is translated as "canal", then as "canard" when you strike the `H`. To access "canard", you will need to strike `KA/NAl/H`. Note that it is possible to strike the H key several times in a row to cycle through the entire list of homophones of the same stroke. As with apostrophes, you will need to install the corresponding plugin: [plover_cycle_homophones](https://github.com/AntoineBalaine/plover_cycle_homophones) with the command:
```bash
plover -s plover_plugins install git+https://github.com/AntoineBalaine/plover_cycle_homophones
```

#### Space Key - Splitting the Previous Stroke
With more than 300 thousand words in the dictionary, it is inevitable that Plover will confuse some, and that some strokes you want to have on separate words will aggregate to form undesirable results. `KI/F*A/S*` is translated as "kiva z", whereas you might want to write "qui vase". A functionality of the [plover_cycle_homophones](https://github.com/AntoineBalaine/plover_cycle_homophones) plugin allows you to split/disaggregate the previous stroke: `*<space>` allows you to split the previous stroke and `<space>` struck within a syllable allows you to split the last two strokes while adding the new syllable.

> NB: this is also a plugin which I wrote for this project.

---
