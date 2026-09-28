# 20240817 BERT

1. Based I have read the book 《BERT实战基础》，the theoretical concepts seem to lay on my basic solid cognition.I want to have a note on how to acquire knowledge on AI and new knowledge. To some extent , I have a vague intuition that brain knowledge is the base of artificial intelligence.Starting with why it was designed this way , I want to dive into the beautiful subject.
2. the guideline about studying BERT and Transformer from chatGPT
    1. understand the basics of Transformers
        
        - Read about Transformers: Attention is All you need. by Vaswani [et.al](http://et.al)
        - focus on understanding self-attention, multi-head attention, positional encoding
    2. learn the BERT model
        
        - BERT (Bidirectional Encoder Representations from Transformers) is built on the Transformer encoder and is pre-trained on large datasets.
        - BERT Paper: “BERT：Pre-training of Deep Bidirectional Transformers for Language Understanding” by Devlin [et.al](http://et.al)
    3. set up my environment
        
        At first, I tried to install transformers on my python 3.9 env.But it failed.Then, I tried to create a new env using conda. And in this environment, I install the needed tools, such as transformers, torch, numpy and pandas.
        

3.Experiment with Pre-trained BERT Models

```java
from transformers import BertTokenizer, BertModel
import torch

# Load pre-trained model to get tokenizer (vocabulary)
tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')

# Encode a sample sentence, and this is the input sentence 
text = "Hello, how are you?"

encoded_input = tokenizer(text, return_tensors='pt')

# Load pre-trained model
model = BertModel.from_pretrained('bert-base-uncased')

# Get the embeddings
with torch.no_grad():
    output = model(**encoded_input)

# Output has the last hidden states.Last_hidden_state is the output from the last layer of the BERT model, representing contextual embeddings for each token in the input sequence.
# The result is torch.Size([1, 8, 768]).
print(output.last_hidden_state.shape)
```

- from transformers import BertTokenizer, BertModel
    
    BertTokenizer is used to convert text into tokens that the BERT model can understand.
    
    BertModel loads the pre-trained BERT model, which will generate embeddings for the tokens.
    
- encoded_input = tokenizer(text, return_tensors='pt')
    
    Tokenizes the input text and converts it into tensor format(’pt’ stands for PyTorch).The meaning of this sentence is to split text into words or subwords, adding special tokens, and converting them into numerical indices.
    
- Explanation
    
    - Tokenizer: Convert the input text into tokens and then into numerical IDs.
    - Model:Process these token IDs to produce embedding
    - Embeddings: The output’s ‘last_hidden_state’ provides these embedding .

4.Based on this code , I have so many specific doubts.what does it mean?

tensor([[[-0.0824, 0.0667, -0.2880, ..., -0.3566, 0.1960, 0.5381], [ 0.0310, -0.1448, 0.0952, ..., -0.1560, 1.0151, 0.0947], [-0.8935, 0.3240, 0.4184, ..., -0.5498, 0.2853, 0.1149], ..., [-0.2812, -0.8531, 0.6912, ..., -0.5051, 0.4716, -0.6854], [-0.4429, -0.7820, -0.8055, ..., 0.1949, 0.1081, 0.0130], [ 0.5570, -0.1080, -0.2412, ..., 0.2817, -0.3996, -0.1882]]])

The answer is above.The tensor obtained from BERT has the shape (batch_size, sequence_length, hidden_size).As the result presents, the shape of this code is {[1, 8, 768]}.

**batch_size** is a very common factor in [AI](http://AI.It) . It represents the number of sequence or sentence in the batch. In my case, there is only one sentence as the input, so the batch_size is 1, meaning **there is only one sentence being processed**.If I provide 2 sentences, the output tensor shape will be (2, sequence_length, hidden_size).**The number of this factor will have an effect on the gradient estimates(to be continued).**

```python
texts = ["Hello, how are you?", "I am fine, thank you!"]
encoded_inputs = tokenizer(texts, padding=True, truncation=True, return_tensors='pt')
output = model(**encoded_inputs)
```

**sequence_length** is a very obscure concept.I spend so much time to understand it. This is the number of tokens in each sentence after tokenization and padding. It represents the number of positions along the sequence of each sentence, and determined by the length of the tokenized input text, padding and truncation setting."Hello," might be split into ["Hello", ","] or similar token pieces.For classification tasks, [CLS] is added at the start, and [SEP] at the end of the sequence.

```python
Detailed Breakdown
Input Text: "Hello, how are you?"

Tokenization: The text is tokenized into a series of tokens. 
The tokenized representation might look like this:
			[CLS] (Special token at the start)
			Hello (Token for "Hello")
			, (Token for the comma)
			how (Token for "how")
			are (Token for "are")
			you (Token for "you")
			[SEP] (Special token at the end)
So, the tokens would be: [CLS], Hello, ,, how, are, you, [SEP].

Token IDs:
{'input_ids': tensor([[ 101,  7592, 117, 1139, 2027, 2017,  102]])}
Here, 101 is the ID for [CLS], 102 is the ID for [SEP], and the other numbers correspond to the tokens.

Tensor Shape:
Batch Size: 1 (because we're processing one text at a time)
Sequence Length: 7 (the number of tokens, including [CLS] and [SEP])
Hidden Size: 768 (the dimension of each token's embedding)
Therefore, the tensor shape would be (1, 7, 768).

Visual Representation
For the input text "Hello, how are you?", the tokenized output tensor can be visualized as:
[
    # Batch Index 0 (only one sentence here)
    [
        [CLS_embedding],     # [CLS] token embedding
        [Hello_embedding],   # "Hello" token embedding
        [Comma_embedding],   # "," token embedding
        [How_embedding],     # "how" token embedding
        [Are_embedding],     # "are" token embedding
        [You_embedding],     # "you" token embedding
        [SEP_embedding]      # [SEP] token embedding
    ]
]
```

The third dimension is **the hidden size**. It is easy to understand.Each token is presented by a 768-dimensional vector.

5.the function of transformer

- **Text Classification**:
    - **Model**: `BertForSequenceClassification` is used to classify text into predefined categories.
    - **Inputs**: The review text is tokenized and passed through the model to get logits, which are converted to probabilities to determine the class.
- **Question Answering**:
    - **Model**: `BertForQuestionAnswering` is used to extract answers from a given passage based on a question.
    - **Inputs**: The question and passage are tokenized together, and the model provides start and end positions for the answer.
- **Text Generation**:
    - **Model**: `GPT2LMHeadModel` is used to generate text based on a given prompt.
    - **Inputs**: The prompt is tokenized and used to generate text sequences. The generated text is decoded from token IDs.