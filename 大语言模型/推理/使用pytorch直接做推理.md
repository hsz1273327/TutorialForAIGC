# 使用pytorch直接做推理

pytorch当然可以用于直接推理,但显然的它只适合用来测试调试.这里不管怎么样也展示下.我们以[Qwen/Qwen2-1.5B-Instruct](https://huggingface.co/Qwen/Qwen2-1.5B-Instruct)为例来演示如何使用pytorch直接做推理.

虽说是用pytorch做推理,但我们一般还是要借助huggingface,它太方便了.我们需要在环境中安装如下依赖

+ pytorch,用它做推理嘛
+ huggingface-cli,用来下载模型,国内建议从镜像站[hf-mirror](https://hf-mirror.com/)下载
+ transformers,huggingface提供的大模型加载调用工具

之后按如下步骤操作

1. 先下载需要的大模型,

    ```bash
    huggingface-cli download --resume-download Qwen/Qwen2-1.5B-Instruct --local-dir ~/WorkSpace/Models/Qwen/Qwen2-1.5B-Instruct
    ```

2. 编写测试程序

    ```python
    from transformers import AutoModelForCausalLM, AutoTokenizer
    device = "cuda"
    # 加载模型
    model = AutoModelForCausalLM.from_pretrained(
        "~/WorkSpace/models/Qwen/Qwen2-1.5B-Instruct",
        torch_dtype="auto",
        device_map="auto"
    )
    tokenizer = AutoTokenizer.from_pretrained("~/WorkSpace/models/Qwen/Qwen2-1.5B-Instruct")
    # 推理
    prompt = "和我说说你是谁."
    messages = [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": prompt}
    ]
    text = tokenizer.apply_chat_template(
        messages,
        tokenize=False,
        add_generation_prompt=True
    )
    print("模板化后："+text)
    model_inputs = tokenizer([text], return_tensors="pt").to(device)
    print(model_inputs.input_ids)
    # Directly use generate() and tokenizer.decode() to get the output.
    # Use `max_new_tokens` to control the maximum output length.
    generated_ids = model.generate(
        model_inputs.input_ids,
        max_new_tokens=512
    )
    generated_ids = [
        output_ids[len(input_ids):] for input_ids, output_ids in zip(model_inputs.input_ids, generated_ids)
    ]
    print("========================================================================")
    response = tokenizer.batch_decode(generated_ids, skip_special_tokens=True)[0]
    print(response)
    ```
