# LZ Training Data Viewer
Reversing LZ training data into commented sgf files

This is my personal repo to share some analysis I have done on Leela Zero training data. As a way to learn to use Github.

Fresh beginner in Python and in object oriented programing, I chose as a toy problem to write a python script that reverses Leela Zero training data into self-play games sgf files, embedding some statistics on the search policy:

![image](https://user-images.githubusercontent.com/37498331/46257726-b7b93a00-c4be-11e8-9a2f-978c3c632e3c.png)

## Methodology

Empirically, I have found that LZ training data files consist in a series of training sample corresponding to successive positions in a series of self-play games. Currently, each training data chunk wraps 32 such games.

By taking advantage of this property (successiveness of positions) and comparin the input planes of a given position with the input planes of the next position, it is possible to deduce the move that was actually picked by temperature (T=1) in that given position. And to check what were the policy value (p_picked) and the rank in the search (picked_rank) for that move. In turn, this allows to reconstruct the whole game and to generate a sfg file with comments on the search policy distribution.

This script also outputs a .csv table in which each line corresponds to a training position, with meta data and the basic statistics:

```
    # Position meta-data:
    chunk_id:          chunk number within training file
    game_id:           game number within the chunk (start at 1)
    game_len_raw:      game length, raw
    game_length:       game length, floored by 20
    move_count_raw:    move count, raw
    move_count:        move count, floored by 20
    side_to_play:      color to play (Black/White)
    played_by:         move belongs to winner/loser of the game
    
    # Policy statistics:
    p_picked_raw       policy of picked move, raw
    p_picked           policy of picked move, floored by 0.02
    picked_rank        the rank of the move picked by temperature in the search policy (best move rank = 1)
    p_pass_raw:        policy probability of pass move
    p_pass:            policy probability of pass move, floored by 0.02
    pass_rank:         the rank of the pass move in the search policy 
    p_max_raw:         policy max (=policy of best move), raw
    p_max:             policy max (=policy of best move), floored by 0.02
    p_best5_raw:       cumulated policy probability of top  5 moves, raw
    p_best5 :          cumulated policy probability of top  5 moves, floored by 0.02
    p_best10_raw:      cumulated policy probability of top 10 moves, raw
    p_best10:          cumulated policy probability of top 10 moves, floored by 0.02
    p_pass_raw:        policy probability of pass move
    p_pass:            policy probability of pass move, floored by 0.02
    norm_cost_raw      normalized computing cost of genrating the next move in current position = tree reuse ratio
    norm_cost          normalized computing cost of genrating the next move in current position, floored by 0.02
    
    # Commented sgf:
    chunk_XXXX_game_YYYY.sgf : a text file in FF4 format of the game ID YYYY of chunk ID XXXX
```

**Sample files:**
[train_dec5b9d8__1000.zip](https://github.com/Ishinoshita/Leela-Zero-Training-Data-Viewer/files/2435705/train_dec5b9d8__1000.zip)

![image](https://user-images.githubusercontent.com/37498331/46316483-947eaf80-c5d0-11e8-9de1-9e48dc87530a.png)


# Preliminery observations on the training data

## Data used

LZ177-generated training data uploaded on 18-Sep-2018 (network hash dec5b9d8, promoted on 15-Sep-2018).

Since I did analysis with Excel (1M lines limit), I took arbitrary slices of 100 consecutive chunks, each corresponding to ~3200 games or ~550k to 600k positions. I analysed chunks 0 to 99, then chunks 1000 to 1099 to check that the conclusions were standing and independent from chunks sampling.

## Found weird data !

I quicly spotted weird policy values in both slices of chunks, not looking like multiples of 1/600th., like here:

![image](https://user-images.githubusercontent.com/37498331/46261875-fb7d6500-c4f9-11e8-8de8-0245f2b93eeb.png)

Suspecting some flaw in my code (although a priori it does not do any arithmetic processing on the values p_picked_raw, p_max_raw and p_pass_raw, just a conversion to float, a sorting step before conversion back to string), I checked the raw training data and found the weird data were already there:

![image](https://user-images.githubusercontent.com/37498331/46261993-7004d380-c4fb-11e8-8b62-de236773cdc7.png)

    0.737846 = 368923/500000, 0.5M not really a reasonable number of visits ...

Note that when the policy contains such a weird value, all non-zero policy values are weird in the training sample. As a shortcut, I here after inaccurately designate such values as *non-fractional*, as they look more like real numbers than like fractions of any reasonable number of visits.

I did an in depth sanity check of the chunks 1000-1099 and found a surprising high proportion of positions containing such non-fractional values: 40592 / 609087, ie 6.7% (not a pee in the ocean ... ).

Then I spotted another kind of weirdness, namely positions where p_picked_raw = p_max_raw = 1 during very long sequence in the game, particularly early in the game. Such games exhibit a clearly different distribution in terms of p_picked_raw or p_best_10_raw:

* To start with, two very standard 300 moves games (left: a no-resign game that probably hit the resign threshold ~100 moves; right a resign game):
![image](https://user-images.githubusercontent.com/37498331/46262164-a8a5ac80-c4fd-11e8-9910-9d9c626efd26.png)
and their sgf:  [chunk_1002_game_7.zip](https://github.com/Ishinoshita/LZ-Training-Data-Viewer/files/2432011/chunk_1002_game_7.zip), [chunk_1004_game_29.zip](https://github.com/Ishinoshita/LZ-Training-Data-Viewer/files/2432012/chunk_1004_game_29.zip)

* Then, two 320 moves games (right: yer another normal no-resign game; left: what I call an out-of-distribution game): 
![image](https://user-images.githubusercontent.com/37498331/46262178-c541e480-c4fd-11e8-8782-589ba62aaa76.png)

Although p_max_raw = 1 may not be strictly impossible, a series of such values seems almost impossible.

I initially suspected fake data. But when I replayed some sfg with Sabaki (LZ177, 1600v, no randomness) I found that LZ177 almost always agreed with the game trajectory. Check [chunk_1042_game_7.zip](https://github.com/Ishinoshita/LZ-Training-Data-Viewer/files/2432014/chunk_1042_game_7.zip).

This second issue is affecting less training samples (1731 / 609087, or 0.28%).

I then discoovered that these two types of weirdness affect the very same games. In such games, either p_picked = p_max = 1 and the rest of the policy if de facto 0, or all values are not multiples of 1/1600th.

Overall, the fraction of training data affected by one or the other of these behaviour is 6.9%.

Since these data doesn't look synthetic (not indentical games; they seems to follow LZ177 inclination), my two cents are that these games are genuine self-play games, but played with a different, much lower temperature, causing non fractional policy values, and p_max = 1 most of the time in the beginning of the game, then dropping a bit, failing to resist to the flattening effect of the search when the game is completely out of balance and the value very high/low.

To complexify a bit, I found that not all games with non-factrional policy exhibit obvious out of distribution policy and p_max = 1 ... Herebelow, for example, all games have all positions with non-fractional policy values (but for the 1 and 0 when p_max=1), but these games look normal, but for game 1003_19 !
![image](https://user-images.githubusercontent.com/37498331/46262441-8c0b7380-c501-11e8-923e-3514e61049ee.png).

Thus, if I'm right with my variable temperature theory, our guy, if he exists, is toying with different temperature values ...

Of course, there might be more simple explanations (bug somewhere in leelaz or autoptp ? A known issue, already spotted ? Eager to know !).

## Correlation between value collapse and policy flattening

*Work in progress*

## Randomness level in no-resign self-play games is still quite impressive !

*Work in progress*
