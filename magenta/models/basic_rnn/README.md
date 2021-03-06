### About Basic RNN

This model provides baselines for the application of language modeling to melody
generation. This code also serves as a working example for implementing a
language model in TensorFlow. In this code, an LSTM cell is used, but any type of cell can be swapped in.

This model takes monophonic melodies, meaning one note plays at a time. Use ```convert_sequences_to_melodies.py``` to extract monophonic melodies from ```NoteSequence``` protos made from your MIDI files. The script will look at each instrument track and extract a melody line if it is at least 7 measures long, and at least 5 unique pitches (with octave equivalence). If multiple notes play at the same time, one note is kept.

### How to Use

First, follow the instructions in our top level [README.md](https://github.com/tensorflow/magenta/blob/master/README.md) for building models and creating a dataset.

Run ```convert_sequences_to_melodies.py``` on the output from ```convert_midi_dir_to_note_sequences.py```, which will extract melodies from the MIDI data. The output is written to disk as a tfrecord file that contains ```SequenceExample``` protos. TensorFlow readers in the ```basic_rnn``` model can read ```SequenceExample``` protos from disk directly into the model. In this example, we create an evaluation dataset in a second tfrecord file, but that can be omitted by leaving out the ```eval_output``` and ```eval_ratio``` flags.

```
# TFRecord file containing NoteSequence protocol buffers from convert_midi_dir_to_note_sequences.py.
SEQUENCES_TFRECORD=/tmp/notesequences.tfrecord

# TFRecord file that TensorFlow's SequenceExample protos will be written to. This is the training dataset.
TRAIN_DATA=/tmp/training_melodies.tfrecord

# Optional evaluation dataset. Also, a TFRecord file containing SequenceExample protos.
EVAL_DATA=/tmp/evaluation_melodies.tfrecord

# Fraction of input data that will be written to the eval dataset (if eval_output flag is set).
EVAL_RATIO=0.10

# Name of the encoder to use. See magenta/lib/encoders.py.
ENCODER=basic_one_hot_encoder

bazel run //magenta/models:basic_rnn_create_dataset -- \
--input=$SEQUENCES_TFRECORD \
--train_output=$TRAIN_DATA \
--eval_output=$EVAL_DATA \
--eval_ratio=$EVAL_RATIO \
--encoder=$ENCODER
```

#### Running training in depth

Build ```basic_rnn_train``` first so that it can be run multiple times in parallel.

```bazel build //magenta/models:basic_rnn_train```

Save train and eval datasets as ```/tmp/training_melodies.tfrecord``` and ```/tmp/eval_melodies.tfrecord```.

Create an experiment directory, say ```/tmp/basic_rnn```, and choose a subdirectory to save this run in, like ```/tmp/basic_rnn/run1```. Increase the number to run2, run3, etc... every time you rerun the same experiment, so that you don't clobber previous experiment output.

Lets create an LSTM model with 1 cell of size 50. So the hyperparameter string is ```'{"rnn_layer_sizes":[50]}'```.

Run training job from the project root

```./bazel-bin/magenta/models/basic_rnn_train --experiment_run_dir=/tmp/basic_rnn/run1 --sequence_example_file=$TRAIN_DATA --eval=false --hparams='{"rnn_layer_sizes":[50]}' --num_training_steps=20000```

Optionally run eval job in parallel with training job

```./bazel-bin/magenta/models/basic_rnn_train --experiment_run_dir=/tmp/basic_rnn/run1 --sequence_example_file=$EVAL_DATA --eval=true --hparams='{"rnn_layer_sizes":[50]}' --num_training_steps=20000```

Run TensorBoard to view training results

```tensorboard --logdir=/tmp/basic_rnn```

Go to [http://localhost:6006](http://localhost:6006) to view TensorBoard dashboard.

#### Run training with script

Alternatively, there is a shell script included for your convenience. Run it from ```magenta/models/basic_rnn```.

```./run_basic_rnn_train.sh $EXPERIMENT_DIR $HYPERPARAMETER_STRING $NUM_TRAINING_STEPS $TRAIN_DATA [$EVAL_DATA]```

Where

* ```$EXPERIMENT_DIR``` is the experiment directory, such as ```/tmp/basic_rnn```
* ```$HYPERPARAMETER_STRING``` is a Python dictionary literal containing hyperparameters, such as ```'{"rnn_layer_sizes":[50]}'```
* ```$NUM_TRAINING_STEPS``` is an integer giving number of training iterations to run, such as ```20000```
* ```$TRAIN_DATA``` is the path to the training dataset (a tfrecord file), such as ```/tmp/training_melodies.tfrecord```
* ```$EVAL_DATA```, an optional argument, is the path to the eval dataset (a tfrecord file), such as ```/tmp/eval_melodies.tfrecord```

This script automatically assigns new folders for new runs in each experiment directory. The eval job will optionally run if the path to the eval dataset is given. This script also runs TensorBoard.

#### Generating melodies

To generate, we need to load a checkpoint of a trained model. Look at your experiment directory, in this example ```/tmp/basic_rnn```. Choose the best run under that experiment, in this example ```/tmp/basic_rnn/run1```. ```basic_rnn_generate``` will look for the most recent checkpoint in that run.

The generator takes a melody as input to prime the model, meaning the model will be fed the primer melody first and then extend it. This primer should be a short monophonic melody stored in a MIDI file. You can make a MIDI file with any MIDI sequencer or digital audio workstation. If you do not have one, there are online sequencers such as [https://onlinesequencer.net/](https://onlinesequencer.net/). We provided an example primer called primer.mid.

The batch size is the number of melodies that will be generated. Lets choose 64.

The generator will take many samples from the model's distribution. You can choose how many outputs will be saved. Here we save 16.

Lets generate 64 new timesteps.

```
# Provide a MIDI file to use as a primer for the generation.
# The MIDI should just contain a short monophonic melody.
# primer.mid is provided as an example.
PRIMER_PATH=<absolute location of your primer MIDI file>

bazel run //magenta/models:basic_rnn_generate -- \
--experiment_run_dir=/tmp/basic_rnn/run1 \
--hparams='{"rnn_layer_sizes":[50]}' \
--primer_midi=$PRIMER_PATH \
--output_dir=/tmp/basic_rnn_generated \
--num_steps=64 \
--num_outputs=16
```
