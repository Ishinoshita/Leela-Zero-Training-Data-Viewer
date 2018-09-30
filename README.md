### LZ Training Data Viewer
Reversing LZ training data into commented sgf files

Fresh beginner in Python and in object oriented programing, I chose as a toy problem to write a python script that reverses Leela Zero training data into sgf files of the self-play games, embedding some statistics on the search policy:

![image](https://user-images.githubusercontent.com/37498331/46257726-b7b93a00-c4be-11e8-9a2f-978c3c632e3c.png)

## Methodology

As far as I can tell by looking at the data (devs may confirm), LZ training data files consists in game positions corresponding to successive positions of a given self-play game. Currently, each training data chunk wraps 32 such games.

By taking advantage of this property (successiveness of positions) it is possible compare the input planes of a given position with the input planes of the next position and to to deduce the move that has actually been picked by temperature (T=1) in that given position. And to check what was policy value (p_picked) and the rank in the search (picked_rank) for that move.

Gaining access to the actual move picked in each position allows to reconstruct the whole game and to generate a sfg file with comments on the search policy distribution. This script also outputs a .csv table in which each line corresponds to a training position, with meta data and the basic statistics.

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
