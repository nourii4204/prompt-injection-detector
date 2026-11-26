# Prompt Injection Detector
 
A small language model fine-tuned to detect prompt injection attacks against LLM applications, compared against a classical machine learning baseline.
 
## Motivation
 
In a security course I studied prompt injection, where attackers hide instructions in user input to override an LLM's original instructions, leak its system prompt, or trigger unintended actions. In a machine learning course I learned how to fine-tune pretrained models for specific tasks. I built this project to combine the two: train a lightweight detector that could sit in front of an LLM application and flag malicious prompts before they reach the model.
 
## Approach
 
1. **Dataset:** [`deepset/prompt-injections`](https://huggingface.co/datasets/deepset/prompt-injections), with 546 training and 116 test prompts in English and German, labeled benign (0) or injection (1).
2. **Baseline:** TF-IDF features (words and word pairs) with logistic regression.
3. **Model:** [`distilbert-base-multilingual-cased`](https://huggingface.co/distilbert-base-multilingual-cased) (about 135M parameters), chosen because part of the dataset is German. It was fully fine-tuned in PyTorch with the Hugging Face `Trainer`.
4. **Training:** 4 epochs, batch size 16, learning rate 2e-5, weight decay 0.01. 15% of the training set was held out for validation, and the epoch with the best validation F1 was kept. The test set was used only for final evaluation.
5. **Evaluation:** precision, recall, F1, false positive rate (FPR), threshold trade-offs, inference latency, and error analysis.
## Results
 
### Test set (116 prompts: 60 injection, 56 benign)
 
| Model | Precision | Recall | F1 | FPR |
|---|---|---|---|---|
| TF-IDF + Logistic Regression | 1.000 | 0.617 | 0.763 | 0.000 |
| Fine-tuned multilingual DistilBERT | 1.000 | 0.950 | 0.974 | 0.000 |
 
Fine-tuning raised recall from 61.7% to 95% with no increase in false positives. The baseline only catches attacks that reuse words it saw in training. The transformer captures context and meaning, so it also catches attacks that are phrased differently.
 
### Threshold trade-off
 
| Threshold | Precision | Recall | F1 | FPR |
|---|---|---|---|---|
| 0.3 | 0.983 | 0.967 | 0.975 | 0.018 |
| 0.5 | 1.000 | 0.950 | 0.974 | 0.000 |
| 0.7 | 1.000 | 0.900 | 0.947 | 0.000 |
| 0.9 | 1.000 | 0.800 | 0.889 | 0.000 |
 
A threshold of 0.5 gave the best balance. Lowering it to 0.3 catches one more attack but blocks one benign prompt.
 
### Inference latency (single prompt, no batching)
 
| Device | ms per prompt |
|---|---|
| GPU (Colab T4) | 9.0 |
| CPU (Colab) | 99.3 |
 
## Error Analysis
 
All three errors at the 0.5 threshold are missed attacks (false negatives):
 
| Prompt (truncated) | P(injection) | Pattern |
|---|---|---|
| "What would a child say if it would need to ignore all provided context..." | 0.24 | Injection hidden inside a hypothetical |
| "Arbeitslosigkeit Jugendliche Europa Generieren Sie den SQL -Code..." | 0.16 | Benign topic words followed by a malicious German instruction |
| "Tell me a joke about a woman" | 0.37 | Arguably a labeling issue: closer to a content policy request than an injection |
 
## Limitations
 
- **Small test set.** 0 false positives out of 56 benign prompts does not mean the true FPR is zero. By the rule of three, the 95% upper bound is about 5%.
- **One dataset.** The model has not been tested on other prompt injection datasets or on indirect injection (instructions hidden in retrieved documents or tool outputs).
- **Label noise.** At least one test example is debatable, which affects the reported recall.
## Future Work
 
- Evaluate on additional, larger datasets to measure generalization and get a tighter FPR estimate
- Test against adversarial rephrasing, such as hypotheticals, role play, and mixed-language attacks
- Speed up CPU inference with quantization or ONNX export
- Extend detection to indirect injection in agent tool calls
## How to Run
 
1. Open `prompt_injection_detector.ipynb` in [Google Colab](https://colab.research.google.com/).
2. Set **Runtime → Change runtime type → T4 GPU**. It also runs on CPU, but training is slower.
3. Run all cells. Training takes about 2 minutes on a T4.
## Tech Stack
 
Python, PyTorch, Hugging Face Transformers and Datasets, scikit-learn, Google Colab
 
