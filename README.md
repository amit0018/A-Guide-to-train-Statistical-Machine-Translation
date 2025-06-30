# A Guide to train MOSES-based Statistical Machine Translation for Source language (ne) to Target Language (hi)

------compile MOSES---------

./bjam -a --with-boost=/home/amit/smt/boost_1_72_0 --with-cmph=/home/amit/smt/cmph-2.0 -j4

-------------SMT Translation Procedure--------

/home/amit/smt/mosesdecoder/scripts/tokenizer/tokenizer.perl -l hi < /home/amit/smt/corpus/train.hi > /home/amit/smt/corpus/train.tok.hi



/home/amit/smt/mosesdecoder/scripts/training/clean-corpus-n.perl /home/amit/smt/corpus/train.tok ne hi /home/amit/smt/corpus/train.clean 1 80




/home/amit/smt/mosesdecoder/bin/lmplz -o 3 </home/amit/smt/corpus/train.tok.hi > train.arpa.hi





/home/amit/smt/mosesdecoder/bin/build_binary train.arpa.hi train.blm.hi



---------TRAINING TRANSLATION SYSTEM---------

mkdir /home/amit/smt/working
cd /home/amit/smt/working
nohup nice /home/amit/smt/mosesdecoder/scripts/training/train-model.perl -root-dir train \
-corpus /home/amit/smt/corpus/train.clean                             \
-f ne -e hi -alignment grow-diag-final-and -reordering msd-bidirectional-fe \
-lm 0:3:/home/amit/smt/lm/train.blm.hi:8                          \
-external-bin-dir /home/amit/smt/mosesdecoder/tools -cores 40 >& training.out &



---------TUNING-------

cd $PWD/working
nohup nice /home/amit/smt/mosesdecoder/scripts/training/mert-moses.pl \
/home/amit/smt/corpus/dev.tok.ne /home/amit/smt/corpus/dev.tok.hi \
/home/amit/smt/mosesdecoder/bin/moses train/model/moses.ini --mertdir /home/amit/smt/mosesdecoder/bin/ \
--decoder-flags="-threads 40" >& mert.out &


-----------TESTING (YOU CAN RUN MOSES)

/home/amit/smt/mosesdecoder/bin/moses -f /home/amit/smt/working/mert-work/moses.ini



mkdir /home/amit/smt/working/binarised-model
cd /home/amit/smt/working

/home/amit/smt/mosesdecoder/bin/processPhraseTableMin -in train/model/phrase-table.gz -nscores 4 -out binarised-model/phrase-table

/home/amit/smt/mosesdecoder/bin/processLexicalTableMin -in train/model/reordering-table.wbe-msd-bidirectional-fe.gz -out binarised-model/reordering-table


cp /home/amit/smt/working/mert-work/moses.ini /home/amit/smt/working/binarised-model/

cd binarised-model/



vi moses.ini
# in the vim editor, type i to start editing mode and edit the following
@1. Change PhraseDictionaryMemory to PhraseDictionaryCompact
@2. Set the path of the PhraseDictionaryCompact feature to point to: /home/amit/smt/working/binarised-model/phrase-table.minphr
@3. Set the path of the LexicalReordering feature to point to: /home/amit/smt/working/binarised-model/reordering-table
@4. Save moses.ini
# to save and quit, type ESC and type wq! followed by ENTER.



-----TESTING AFTER COMPACT (BINARIZE THE MODULE)

/home/amit/smt/mosesdecoder/bin/moses -f /home/amit/smt/working/binarised-model/moses.ini


cd /home/amit/smt/working

/home/amit/smt/mosesdecoder/scripts/training/filter-model-given-input.pl filtered-corpus-mini mert-work/moses.ini /home/amit/smt/corpus/dev.tok.ne -Binarizer /home/amit/smt/mosesdecoder/bin/processPhraseTableMin


-------TRANSLATION TEST--------

nohup nice /home/amit/smt/mosesdecoder/bin/moses            \
   -f /home/amit/smt/working/filtered-corpus-mini/moses.ini   \
   < /home/amit/smt/corpus/dev.tok.ne                \
   > /home/amit/smt/working/test.translated.hi         \
   2> /home/amit/smt/working/test.out

------BLEU SCRIPT--------

/home/amit/smt/mosesdecoder/scripts/generic/multi-bleu.perl \
   -lc /home/amit/smt/corpus/dev.tok.hi              \
   < /home/amit/smt/working/test.translated.hi
