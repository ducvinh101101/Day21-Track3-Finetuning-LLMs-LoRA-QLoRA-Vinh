# Lab 21 - Evaluation Report

**Hoc vien**: Mai Duc Vinh - 2A202600587  
**Ngay nop**: 2026-06-25  
**Submission option**: A - Lightweight ZIP

## 1. Setup

- **Base model**: `unsloth/Qwen2.5-3B-bnb-4bit`
- **Dataset**: `5CD-AI/Vietnamese-alpaca-gpt4-gg-translated`
- **Samples**: 200 total, 180 train, 20 eval
- **Format**: Alpaca-style `instruction`, `input`, `output`
- **GPU**: Tesla T4
- **max_seq_length**: 1024
- **Training cost estimate**: about $0.07 at $0.35/hour
- **LoRA target modules**: `q_proj`, `v_proj`
- **Training setup**: QLoRA 4-bit, Unsloth, TRL `SFTTrainer`, 3 epochs, learning rate `2e-4`, cosine scheduler, effective batch size 8

## 2. Rank Experiment Results

| Rank | Alpha | Trainable Params | Train Time (min) | Peak VRAM (GB) | Eval Loss | Perplexity |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 8 | 16 | 1,843,200 | 4.01 | 7.22 | 1.5577 | 4.7479 |
| 16 | 32 | 3,686,400 | 4.12 | 6.62 | 1.5161 | 4.5544 |
| 64 | 128 | 14,745,600 | 4.02 | 8.00 | 1.4768 | 4.3790 |

**Note**: Base-model perplexity was not computed in this run. The qualitative comparison still includes base vs fine-tuned outputs, but the quantitative table only compares the three trained LoRA ranks.

## 3. Loss Curve Analysis

For the `r=16` baseline, the training loss generally decreased from about `1.61` at step 5 to about `1.39` at step 65. The curve is not perfectly smooth, with small increases around steps 35, 40, 55, and 60, but the overall trend is downward. This suggests that the model learned from the dataset instead of failing to train.

Mid-training evaluation was disabled to fit the T4 memory budget, so overfitting cannot be fully confirmed from eval loss over time. However, final eval perplexity improved as rank increased: `r=8` reached `4.7479`, `r=16` reached `4.5544`, and `r=64` reached `4.3790`. This pattern suggests that higher rank gave the adapter more capacity and improved evaluation performance on this small eval split.

## 4. Qualitative Comparison

| Prompt | Observation |
| --- | --- |
| Explain machine learning for beginners | The fine-tuned output is more explanatory and beginner-oriented, though both outputs are usable. |
| Write Python code for Fibonacci | The fine-tuned output starts with a clearer structure, but the code shown in the truncated sample still needs manual checking because it contains an odd Vietnamese error message. |
| List 5 UI/UX principles | The fine-tuned output is more natural in Vietnamese and gives fuller explanations, while the base output is shorter and repetitive. |
| Summarize LoRA vs QLoRA | Both outputs contain some conceptual inaccuracies, but the fine-tuned answer attempts a more structured comparison. This is a weak example and should be treated as only partially improved. |
| Compare prompt engineering, RAG, and fine-tuning | The fine-tuned response is more verbose, but it also uses some imprecise wording. The improvement is mixed. |

Overall, the qualitative results show that the fine-tuned model often produces longer Vietnamese explanations and more complete answer formats. However, it does not guarantee factual correctness, especially for technical prompts. This matches the lesson that fine-tuning is useful for style, format, and behavior, while factual grounding should still rely on evaluation and possibly RAG.

## 5. Conclusion ve Rank Trade-off

Trong thí nghiệm này, `r=64` cho perplexity tốt nhất với giá trị `4.3790`, thấp hơn `r=16` là `4.5544` và `r=8` là `4.7479`. Điều này cho thấy tăng rank giúp adapter có nhiều tham số học hơn và biểu diễn được nhiều thay đổi hơn so với base model. Tuy nhiên, `r=64` cũng có số trainable params lớn hơn nhiều, khoảng `14.7M`, so với `3.7M` của `r=16` và `1.8M` của `r=8`. Peak VRAM của `r=64` cũng cao nhất, khoảng `8.00 GB`.

Nếu chỉ nhìn perplexity, `r=64` là tốt nhất. Nhưng nếu cân bằng giữa chất lượng, chi phí bộ nhớ và kích thước adapter, `r=16` là lựa chọn hợp lý hơn cho production hoặc bài lab trên T4. `r=16` cải thiện rõ so với `r=8`, trong khi vẫn nhỏ hơn và nhẹ hơn nhiều so với `r=64`. Với dataset nhỏ 200 samples, khác biệt perplexity giữa `r=16` và `r=64` không quá lớn, nên `r=16` có ROI tốt hơn. Nếu cần chất lượng tối đa và có đủ GPU/storage, có thể chọn `r=64`; còn nếu cần adapter gọn, dễ deploy, tôi sẽ chọn `r=16`.

## 6. What I Learned

- LoRA rank ảnh hưởng trực tiếp tới số trainable parameters: rank càng cao thì adapter càng có nhiều capacity, nhưng cũng tốn VRAM và storage hơn.
- Perplexity giúp so sánh định lượng giữa các rank, nhưng qualitative examples vẫn rất quan trọng vì output có thể dài hơn mà chưa chắc đúng hơn.
- Fine-tuning không thay thế được factual grounding. Với các câu hỏi kỹ thuật như LoRA/QLoRA, model vẫn có thể trả lời sai hoặc mơ hồ, nên cần evaluation cẩn thận và có thể kết hợp RAG khi cần kiến thức chính xác.
