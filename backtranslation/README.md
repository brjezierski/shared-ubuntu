# Backtranslation from hsb to dsb
Backtranslation should expand the bilingual dsb-hsb corpus to retrain the dsb-hsb translation model, transliteration mining model and transliteration model.

## Steps
1. Use the baseline hsb-dsb machine translation model to generate parallel dsb data form full_monolingual.hsb file.
  ```
  nohup nice ~/mosesdecoder/bin/moses -f ~/sorbian-transliteration/baseline-dsb-hsb/model/moses.ini \
    < ~/sorbian-transliteration/data/hsb/clean/full_monolingual.hsb > ~/sorbian-transliteration/backtranslation/full_monolingual.dsb 2> trace.out
  ```
  
2. Append the parallel hsb-dsb train data with the results of step 1.
  ```
  cat ~/sorbian-transliteration/backtranslation/full_monolingual.dsb ~/sorbian-transliteration/data/hsb-dsb/corpus/clean/train.dsb > ~/sorbian-transliteration/backtranslation/train.dsb
  cat ~/sorbian-transliteration/data/hsb/clean/full_monolingual.hsb ~/sorbian-transliteration/data/hsb-dsb/corpus/clean/train.hsb > ~/sorbian-transliteration/backtranslation/train.hsb
  ```
  
3. Retrain the dsb-hsb translation model with the transliteration mining and model.

  ```
  ~/mosesdecoder/scripts/training/train-model.perl \
    -root-dir ~/sorbian-transliteration/backtranslation/ \
    --corpus ~/sorbian-transliteration/backtranslation/train \
    --f dsb \
    --e hsb \
    -external-bin-dir ~/sorbian-transliteration/external_bin \
    -mgiza --last-step=3
  ```
  ```
  ~/mosesdecoder/scripts/Transliteration/train-transliteration-module.pl \
    --corpus-f ~/sorbian-transliteration/backtranslation/train.dsb \
    --corpus-e ~/sorbian-transliteration/backtranslation/train.hsb \
    --alignment ~/sorbian-transliteration/backtranslation/model/aligned.grow-diag-final \
    --moses-src-dir ~/mosesdecoder \
    --external-bin-dir ~/sorbian-transliteration/external_bin \
    --input-extension dsb \
    --output-extension hsb \
    --out-dir ~/sorbian-transliteration/backtranslation/transliteration-model \
    --srilm-dir ~/srilm/bin/aarch64
  ```
  
  ```
  nohup nice ~/mosesdecoder/scripts/training/train-model.perl \
    -root-dir ~/sorbian-transliteration/backtranslation \
    -corpus ~/sorbian-transliteration/backtranslation/train \
    -f dsb -e hsb -alignment grow-diag-final-and \
    -reordering msd-bidirectional-fe -lm 0:3:$HOME/sorbian-transliteration/backtranslation/transliteration-model/lm/targetLM:8 \
    -external-bin-dir ~/sorbian-transliteration/external_bin \
    -post-decoding-translit yes \
    -transliteration-phrase-table ~/sorbian-transliteration/backtranslation/transliteration-model/model/phrase-table.gz >& training.out &
  ```
  
4. Get translations and transliterations of the test set.

  ```
  nohup nice ~/mosesdecoder/bin/moses -f ~/sorbian-transliteration/backtranslation/model/moses.ini \
    -output-unknowns ~/sorbian-transliteration/backtranslation/oov.dsb \
    < ~/sorbian-transliteration/data/hsb-dsb/corpus/clean/test.dsb > ~/sorbian-transliteration/backtranslation/results/test.translated.hsb 2> trace.out
  ```
  
  ```
  ~/mosesdecoder/scripts/Transliteration/post-decoding-transliteration.pl \
    --moses-src-dir ~/mosesdecoder \
    --external-bin-dir ~/sorbian-transliteration/external_bin \
    --transliteration-model-dir ~/sorbian-transliteration/backtranslation/transliteration-model \
    --oov-file ~/sorbian-transliteration/backtranslation/oov.dsb \
    --input-file ~/sorbian-transliteration/backtranslation/results/test.translated.hsb \
    --output-file ~/sorbian-transliteration/backtranslation/results/test.translated.transliterated.hsb \
    --input-extension dsb --output-extension hsb \
    --language-model ~/sorbian-transliteration/backtranslation/transliteration-model/lm/targetLM \
    --decoder ~/mosesdecoder/bin/moses
  ```
  
5. Evaluate the BLEU scores.
  
  ```
  ~/mosesdecoder/scripts/generic/multi-bleu-detok.perl \
    -lc ~/sorbian-transliteration/data/hsb-dsb/corpus/clean/test.hsb \
    < ~/sorbian-transliteration/backtranslation/results/test.translated.hsb;
  ~/mosesdecoder/scripts/generic/multi-bleu-detok.perl \
    -lc ~/sorbian-transliteration/data/hsb-dsb/corpus/clean/test.hsb \
    < ~/sorbian-transliteration/backtranslation/results/test.translated.transliterated.hsb
  ```
