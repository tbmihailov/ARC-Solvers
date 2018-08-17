# ARC-Solvers
Library of baseline solvers for AI2 Reasoning Challenge (ARC) Set (http://data.allenai.org/arc/).
These solvers retrieve relevant sentences from a large text corpus (ARC_Corpus.txt in the
dataset), and use two types of models to predict the correct answer.
 1. An entailment-based model that computes the entailment score for each `(retrieved sentence,
 question+answer choice as an assertion)` pair and scores each answer choice based on the
 highest-scoring sentence.
 2. A reading comprehension model (BiDAF) that converts the retrieved sentences into a paragraph
 per question. The model is used to predict the best answer span and each answer choice is scored
  based on the overlap with the predicted span.
 
 ## Setup environment
 1. Create the `arc_solvers` environment using Anaconda
 
   ```
   conda create -n arc_solvers python=3.6
   ```
 
 2. Activate the environment
 
   ```
   source activate arc_solvers
   ```
 
 3. Install the requirements in the environment: 
 
   ```
   sh scripts/install_requirements.sh
   ```
 
 4. Install pytorch as per instructions on <http://pytorch.org/>. Command as of Feb. 26, 2018:
 
   ```
   conda install pytorch torchvision -c pytorch
   ```
  

 ## Setup data/models
 1. Download the data and models into `data/` folder. This will also build the ElasticSearch
 index (assumes ElasticSearch 6+ is running on `ES_HOST` machine defined in the script)
  ```
  sh scripts/download_data.sh
  ```

 2. Download and prepare embeddings. This will download glove.840B.300d.zip from https://nlp.stanford.edu/projects/glove/ and 
 convert it to glove.840B.300d.txt.gz which is readable from AllenNLP
   ```
  sh download_and_prepare_glove.sh
  ```

 
## Running baseline models
Run the entailment-based baseline solvers against a question set using `scripts/evaluate_solver.sh`

### Running a pre-trained DGEM model
For example, to evaluate the DGEM model on the Challenge Set, run:
```
sh scripts/evaluate_solver.sh \
	data/ARC-V1-Feb2018/ARC-Challenge/ARC-Challenge-Test.jsonl \
	data/ARC-V1-Models-Aug2018/dgem/
```
  Change `dgem` to `decompatt` to test the Decomposable Attention model.

### Running a pre-trained BiDAF model
To evaluate the BiDAF model, use the `evaluate_bidaf.sh` script
```
 sh scripts/evaluate_bidaf.sh \
    data/ARC-V1-Feb2018/ARC-Challenge/ARC-Challenge-Test.jsonl \
    data/ARC-V1-Models-Aug2018/bidaf/
```

### Training and evaluating the BiLSTM Max-out with Question to Choices Max Attention

This model implements an attention interaction between question and choice context-encoded representations:

The basic outline of this model is to get an embedded representation for the
question and choice, model an interaction between them and use a linear layer to het an attention score.
Pseudo code of the model:

```python
# encode question and each choice
question_encoded = context_enc(question_words)  # context_enc can be any AllenNLP supported or None. Bi-directional LSTM is used
choice_encoded = context_enc(choice_words)  # `length X hidden_size`

#get a single vector representations for question and choice
question_aggregate = aggregate_method(question_encoded)  # aggregate_method can be max, min, avg. ``max`` is used.
choice_aggregate = aggregate_method(choice_encoded)  # `length X hidden_size`

# interaction representaiton
q_to_ch_interaction_repr = concat([question_aggregate, choice_aggregate, choice_aggregate - question_aggregate, question_aggregate
 * choice_aggregate)  # `4 x hidden_size`

# question to choice attention
att_q_to_ch = linear_layer(q_to_ch_interaction_repr)  # the output is a scalar value (size 1) for each question-to-choice interaction

# The `choice_to_question_attention` of the four choices are normalized using ``softmax``
# and the choice with the highest attention is selected as the answer.
answer_id = argmax(softmax([att_q_to_ch0, att_q_to_ch1, att_q_to_ch2, att_q_to_ch3]))

```

The model is inspired by the BiLSTM Max-Out model from Conneau, A. et al. (2017) ‘Supervised Learning of
Universal Sentence Representations from Natural Language Inference Data’.


To train the model, you need to have the data and embeddings downloaded (Step 2. of *Setup data/models* above).

Evaluate the trained model:
```bash
python arc_solvers/run.py evaluate --archive_file data/ARC-V1-Models-Aug2018/max_att/model.tar.gz --evaluation_data_file data/ARC-V1-Feb2018/ARC-Challenge/ARC-Challenge-Test.jsonl
```

or

Train a new model:
```bash
python arc_solvers/run.py train -s trained_models/qa_multi_question_to_choices/serialization/ arc_solvers/training_config/qa/multi_choice/reader_qa_multi_choice_max_att_ARC_Chellenge_full.json
```


## Running against a new question set

 To run the baseline solvers against a new question set, create a file using the JSONL format.
 For example:
 ```
 {
    "id":"Mercury_SC_415702",
    "question": {
       "stem":"George wants to warm his hands quickly by rubbing them. Which skin surface will
               produce the most heat?",
       "choices":[
				  {"text":"dry palms","label":"A"},
				  {"text":"wet palms","label":"B"},
				  {"text":"palms covered with oil","label":"C"},
				  {"text":"palms covered with lotion","label":"D"}
                 ]
    },
    "answerKey":"A"}
 ```
  Run the evaluation scripts on this new file using the same commands as above.


 ## Running a new Entailment-based model
  To run a new entailment model (implemented using AllenNLP), you need to
   1. Create a `Predictor` that converts the input JSON to an `Instance` expected by your
   entailment model. See [DecompAttPredictor](arc_solvers/service/predictors/decompatt_qa_predictor.py)
   for an example.

   2. Add your custom predictor to the [predictor overrides](arc_solvers/commands/__init__.py#L8)
   For example, if your new model is registered using `my_awesome_model` and the predictor is
   registered using `my_awesome_predictor`, add `"my_awesome_model": "my_awesome_predictor"` to
   the `predictor_overrides`.

   3. Run the `evaluate_solver.sh` script with your learned model in `my_awesome_model/model.tar.gz`:

    ```bash
     sh scripts/evaluate_solver.sh \
        data/ARC-V1-Feb2018/ARC-Challenge/ARC-Challenge-Test.jsonl \
        my_awesome_model/
    ```

## Running a new Reading Comprehension model
 To run a new reading comprehension (RC) model (implemented using AllenNLP), you need to
   1. Create a `Predictor` that converts the input JSON to an `Instance` expected by your
   RC model. See [BidafQaPredictor](arc_solvers/service/predictors/bidaf_qa_predictor.py)
   for an example.

   2. Add your custom predictor to the [predictor overrides](arc_solvers/commands/__init__.py#L8)
   For example, if your new model is registered using `my_awesome_model` and the predictor is
   registered using `my_awesome_predictor`, add `"my_awesome_model": "my_awesome_predictor"` to
   the `predictor_overrides`.

   3. Run the `evaluate_bidaf.sh` script with your learned model in `my_awesome_model/model.tar.gz`:

    ```bash
     sh scripts/evaluate_solver.sh \
        data/ARC-V1-Feb2018/ARC-Challenge/ARC-Challenge-Test.jsonl \
        my_awesome_model/
    ```


