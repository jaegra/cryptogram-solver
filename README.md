# cryptogram-solver

Solver for [cryptograms](https://en.wikipedia.org/wiki/Cryptogram) (substitution ciphers).

![](references/demo.gif)

# Table of contents

- [Arguments](#args)
- [Examples](#examples)
- [What is a cryptogram?](#whatis)
- [Method](#method)
  - [Tokenizer](#tokenizer)
  - [Scoring](#scoring)
  - [Optimization](#optimization)
- [Data](#data)
  - [Using the default data](#default)
  - [Using custom data](#custom)
- [Dependencies](#dependencies)
- [TODOs](#todos)


<a name="args"/>

# Arguments

See help: `python solver.py -h`

```
usage: solver.py [-h] [-i NUM_ITERS] [-c CHAR_NGRAM_RANGE CHAR_NGRAM_RANGE]
                 [-w WORD_NGRAM_RANGE WORD_NGRAM_RANGE]
                 [--freqs_path FREQS_PATH] [--docs_path DOCS_PATH] [-n N_DOCS]
                 [-b VOCAB_SIZE] [-p PSEUDO_COUNT]
                 [--log_temp_start LOG_TEMP_START]
                 [--log_temp_end LOG_TEMP_END] [--lamb_start LAMB_START]
                 [--lamb_end LAMB_END] [-l] [-s] [-v]
                 text

positional arguments:
  text                  text to be decrypted

optional arguments:
  -h, --help            show this help message and exit
  -i NUM_ITERS, --num_iters NUM_ITERS
                        number of iterations during simulated annealing
                        process
  -c CHAR_NGRAM_RANGE CHAR_NGRAM_RANGE, --char_ngram_range CHAR_NGRAM_RANGE CHAR_NGRAM_RANGE
                        range of character n-grams to use in tokenization
  -w WORD_NGRAM_RANGE WORD_NGRAM_RANGE, --word_ngram_range WORD_NGRAM_RANGE WORD_NGRAM_RANGE
                        range of word n-grams to use in tokenization
  --freqs_path FREQS_PATH
                        path to word n-gram frequencies (a CSV file) for
                        fitting solver
  --docs_path DOCS_PATH
                        path to corpus (a text file) for fitting solver
  -n N_DOCS, --n_docs N_DOCS
                        number of documents used to estimate token frequencies
  -b VOCAB_SIZE, --vocab_size VOCAB_SIZE
                        size of vocabulary to use for scoring
  -p PSEUDO_COUNT, --pseudo_count PSEUDO_COUNT
                        number added to all token frequencies for smoothing
  --log_temp_start LOG_TEMP_START
                        log of initial temperature
  --log_temp_end LOG_TEMP_END
                        log of final temperature
  --lamb_start LAMB_START
                        Poisson lambda for number of additional letter swaps
                        in beginning; use 0 for single swaps
  --lamb_end LAMB_END   Poisson lambda for number of additional letter swaps
                        at end; use 0 for single swaps
  -l, --load_solver     load pre-fitted solver
  -s, --save_solver     save fitted solver for use later
  -v, --verbose         verbose output for showing solve process
```


<a name="examples"/>

# Examples

The default settings tend to work well most of the time. Usually you only need to specify the encrypted text, saving (`-s`), loading (`-l`) and the number of iterations (`-i`, which you can set to be lower for longer decryptions).

To fit a solver on 1000 documents (`-n 1000`) from a custom corpus (`--docs_path <PATH>`) with a tokenizer that uses character bigrams and trigrams (`-c 2 3`) and word unigrams (`-w 1 1`) with a max vocab size of 5000 (`-b 5000`), save it to file (`-s`) (right now just to `models/cached/`), and solve the encrypted text (represented here as `<ENCRYPTED TEXT>`):

    python solver.py <ENCRYPTED TEXT> -n 1000 --docs_path <PATH> -c 2 3 -w 1 1 -b 5000 -s

To load a fitted solver (`-l`) and run the optimizer for 5000 iterations (`-i 5000`) with a starting lambda (for character swaps; see below) of 1 (`--lamb_start 1`) and verbose output (`-v`):

    python solver.py <ENCRYPTED TEXT> -l -i 5000 --lamb_start 1 -v


<a name="whatis"/>

# What is a cryptogram?

Let's go with [Wikipedia's definition](https://en.wikipedia.org/wiki/Cryptogram):

> A cryptogram is a type of puzzle that consists of a short piece of encrypted text. Generally the cipher used to encrypt the text is simple enough that the cryptogram can be solved by hand. Frequently used are substitution ciphers where each letter is replaced by a different letter or number. To solve the puzzle, one must recover the original lettering. Though once used in more serious applications, they are now mainly printed for entertainment in newspapers and magazines.

For example, let's say you're given the puzzle below.

> "SNDVSODTSBO LDF VSHYO TB NDO TB EBNRYOFDTY KSN PBX LKDT KY SF OBT, DOC D FYOFY BP KZNBX LDF RXBHSCYC TB EBOFBJY KSN PBX LKDT KY SF." -BFEDX LSJCY
 
The goal is to realize that _i_'s were replaced with _S_'s, _m_'s with _N_'s, and so on for all the letters of the alphabet. Once you make all the correct substitutions, you get the following text.

> "Imagination was given to man to compensate him for what he is not, and a sense of humor was provided to console him for what he is." -Oscar Wilde

By hand, you'd use heuristics to solve cryptograms iteratively. E.g., if you see _ZXCVB'N_, you might guess that _N_ is _t_ or _s_. You might also guess that if _X_ appears a lot in the text, it might be a common letter like _e_.

But having a computer solve this is tricky.  You can't brute-force your way through a cryptogram since there are 26! = 403,291,461,126,605,635,584,000,000 different mappings. (That is, the letter _a_ could map to one of 26 letters, _b_ could map to 25, and so on.) And there isn't a surefire way to tell if you've found the correct mapping.


<a name="method"/>

# Method


<a name="tokenizer"/>

## Tokenizer

I use a tokenizer that can generate both character n-grams and word n-grams. The code example below shows how the tokenizer creates character bigrams and trigrams as well as word unigrams.

```python
>>> from cryptogram_solver import solver
>>> tk = solver.Tokenizer(char_ngram_range=(2, 3), word_ngram_range=(1, 1))
>>> tokens = tk.tokenize('Hello!')
>>> print(*tokens, sep='\n')
Token(ngrams=('hello',), kind='word', n=1)
Token(ngrams=('<', 'h'), kind='char', n=2)
Token(ngrams=('h', 'e'), kind='char', n=2)
Token(ngrams=('e', 'l'), kind='char', n=2)
Token(ngrams=('l', 'l'), kind='char', n=2)
Token(ngrams=('l', 'o'), kind='char', n=2)
Token(ngrams=('o', '>'), kind='char', n=2)
Token(ngrams=('<', 'h', 'e'), kind='char', n=3)
Token(ngrams=('h', 'e', 'l'), kind='char', n=3)
Token(ngrams=('e', 'l', 'l'), kind='char', n=3)
Token(ngrams=('l', 'l', 'o'), kind='char', n=3)
Token(ngrams=('l', 'o', '>'), kind='char', n=3)
```


<a name="scoring"/>

## Scoring

To score decryptions, I use the negative log likelihood of the token probabilities. Token probabilities are computed by token "type" (word/character and n-gram count combination). For example, to compute the probability of a character bigram, I divide the frequency of that bigram by the total number of character bigrams. This is done when "fitting" the solver to data (see `Solver.fit()`).

You can think of score as error, so lower is better.


<a name="optimization"/>

## Optimization

I use a modified version of [simulated annealing](https://en.wikipedia.org/wiki/Simulated_annealing) for the optimization algorithm. The algorithm is run for a pre-defined number of iterations, where in each iteration it swaps random letters in the mapping and re-scores the text. It uses softmax on the difference of the scores of the current text and new text to determine whether it wants to keep the new mapping. Note that it's open to accepting worse mappings for the sake of exploration and escaping local minima. Over the course of the optimization, it decreases temperature (exponentially) so that it's decreasingly likely to accept mappings that hurt the score. It also decreases the number of swaps per iteration (probabilistically, using the Poisson process described below) to encourage exploration in the beginning and fine tuning at the end.

My intuition tells me that character n-grams do the heavy lifting for most of the optimization, while the word n-grams help the algorithm "lock in" on good mappings at the end.

```python
def simulated_annealing(encrypted, num_iters):
    """Python-style pseudo(-ish)code for simulated annealing algorithm."""
    best_mapping = Mapping()
    best_score = compute_score(encrypted)

    for i in range(num_iters):
        temp = temp_list[i]  # from scheduler
        num_swaps = swap_list[i]  # from scheduler

        mapping = best_mapping.random_swap(num_swaps)
        text = mapping.translate(encrypted)
        score = compute_score(text)

        score_change = score - best_score

        if exp(-score_change / temp) > uniform(0, 1):  # softmax
            best_mapping = mapping
            best_score = score

    decrypted = best_mapping.translate(encrypted)
    return best_mapping, decrypted
```

To decrease the number of swaps over time, I use a Poisson distribution with a lambda parameter that decreases linearly. The scheduler for the number of swaps is

```python
def schedule_swaps(lamb_start, lamb_end, n):
    for l in linspace(lamb_start, lamb_end, n):
        yield rpoisson(l) + 1 
```

where `lamb_start` is the starting lambda (at the beginning of the optimization), `lamb_end` is the ending lambda, `n` is the number of iterations in the optimization, `linspace` is a function that returns evenly spaced numbers over a specified interval (a basic version of [numpy's implementation](https://docs.scipy.org/doc/numpy/reference/generated/numpy.linspace.html)), and `rpoisson` is a function that draws a random sample from a Poisson distribution given a lambda parameter. Note that a lambda greater than zero gives you *additional* swaps; if `lamb_start` and `lamb_end` are both zero, then in every iteration you swap only once.


<a name="data"/>

# Data

The solver can be fitted using either a corpus of documents (as a text file) or a list of precomputed word unigram frequencies (as a CSV file). It's much quicker to train on precomputed unigram frequencies. However, word bigram frequencies can't be determined from word unigram frequencies, so the highest possible degree in your word n-gram range is 1 if fitting on the unigram frequencies. Therefore, if the user does not choose word n-grams higher than degree 1, then the solver is fitted on the pre-computed unigrams (see `FREQS_PATH` in `defaults.py`); otherwise, it's fitted on the corpus directly (`CORPUS_PATH`).


<a name="default"/>

## Using the default data
If you'd like to use the default data to fit the solver, then download the data [here](https://www.kaggle.com/snapcrack/all-the-news) and put the zip files into the following directory structure and then run `make_data/make_kaggle_news_data.py`. This is a manual process because you need to create a Kaggle account to access the data.

```
data
|-- raw
    |-- corpora
        |-- kaggle_news
            |-- articles1.csv.zip
            |-- articles2.csv.zip
            |-- articles3.csv.zip
```

<a name="custom"/>

## Using custom data
To use your own custom data to fit the solver, you can do one of the following:
- **Change the default paths.** If you have a corpus but not unigram frequencies, then modify `CORPUS_PATH` (where you saved the corpus data) and `FREQS_PATH` (where you want the unigram frequency data to be generated) in `defaults.py`. Then run `corpus_to_freqs.py`. If you have both corpus data and unigram frequency data, then just modify the paths just mentioned.
- **Specify paths each time.** That is, specify either `--freqs_path` or `--docs_path` each time you fit the solver.


<a name="dependencies"/>

# Dependencies

- [tqdm](https://github.com/tqdm/tqdm) for progress bars


<a name="todos"/>

# TODOS
1. Store data somewhere and write script to download it. (And possibly change the data source.)
1. Use random search to find better set of default parameters.
1. See if pre-computing log probabilities results in a significant speedup.
1. See if turning tokens into joined strings speeds things up.
