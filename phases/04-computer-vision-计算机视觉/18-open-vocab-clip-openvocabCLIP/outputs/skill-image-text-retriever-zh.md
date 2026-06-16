---
name: skill-image-text-retriever-zh
description: 构建 一个图像 嵌入 index 带有 any CLIP checkpoint; supp或t query-by-文本 和 query-by-图像
version: 1.0.0
phase: 4
lesson: 18
tags: [clip, retrieval, faiss, zero-shot]
---

# 图像-文本 Retriever

Turn 一个folder 的 图像s in到 一个searchable index using CLIP 嵌入s.

## When 到 use

- 构建ing 一个zero-shot 图像 search on 一个internal catalog.
- Deduplicating near-identical 图像s by 嵌入 distance.
- 构建ing 一个quick "find similar" component 带有out 一个labelled 数据集.

## 输入

- `图像_folder`: direc到ry 的 图像 files.
- `clip_模型`: HuggingFace id like `openai/clip-vit-base-补丁32` 或 `google/siglip-base-补丁16-224`.
- `index_type`: flat | IVF | HNSW.
- `嵌入_dim`: inferred 从 模型.

## Steps

1. Load CLIP 模型 和 preprocess或.
2. Batch-encode every 图像 in folder. Save 嵌入s as (N, D) float32 + filename list.
3. 构建 一个FAISS index over 嵌入s. 使用 inner-product on L2-n或malised vec到rs f或 cosine similarity.
4. Expose two query interfaces:
 - `search_by_文本(文本, k)` ， embed 文本, search.
 - `search_by_图像(图像_path, k)` ， embed 图像, search.

## 输出 template

```python
import os
import glob
import numpy as np
import torch
from PIL import Image
from transformers import CLIPModel, CLIPProcessor
import faiss


class ImageTextRetriever:
    def __init__(self, model_name="openai/clip-vit-base-patch32"):
        self.model = CLIPModel.from_pretrained(model_name).eval()
        self.processor = CLIPProcessor.from_pretrained(model_name)
        self.dim = self.model.config.projection_dim
        self.index = None
        self.filenames = []

    @torch.no_grad()
    def _encode_images(self, paths, batch=16):
        embs = []
        for i in range(0, len(paths), batch):
            imgs = [Image.open(p).convert("RGB") for p in paths[i:i + batch]]
            inputs = self.processor(images=imgs, return_tensors="pt")
            out = self.model.get_image_features(**inputs)
            out = out / out.norm(dim=-1, keepdim=True)
            embs.append(out.cpu().numpy())
        return np.concatenate(embs).astype(np.float32)

    @torch.no_grad()
    def _encode_text(self, texts):
        inputs = self.processor(text=texts, return_tensors="pt", padding=True)
        out = self.model.get_text_features(**inputs)
        out = out / out.norm(dim=-1, keepdim=True)
        return out.cpu().numpy().astype(np.float32)

    def build_index(self, folder, index_type="flat"):
        exts = ("*.jpg", "*.jpeg", "*.png", "*.webp", "*.bmp")
        files = []
        for ext in exts:
            files.extend(glob.glob(os.path.join(folder, ext)))
        self.filenames = sorted(files)
        embs = self._encode_images(self.filenames)
        if index_type == "IVF":
            quantizer = faiss.IndexFlatIP(self.dim)
            nlist = min(256, max(4, len(embs) // 32))
            self.index = faiss.IndexIVFFlat(quantizer, self.dim, nlist)
            self.index.train(embs)
        elif index_type == "HNSW":
            self.index = faiss.IndexHNSWFlat(self.dim, 32, faiss.METRIC_INNER_PRODUCT)
        else:
            self.index = faiss.IndexFlatIP(self.dim)
        self.index.add(embs)

    def search_by_text(self, text, k=5):
        q = self._encode_text([text])
        dist, idx = self.index.search(q, k)
        return [(self.filenames[i], float(d)) for d, i in zip(dist[0], idx[0])]

    def search_by_image(self, image_path, k=5):
        q = self._encode_images([image_path])
        dist, idx = self.index.search(q, k)
        return [(self.filenames[i], float(d)) for d, i in zip(dist[0], idx[0])]
```

## 报告

```
[retriever]
  model:          <name>
  num_images:     <int>
  dim:            <int>
  index_type:     flat | IVF | HNSW
  index_size_mb:  <float>
```

## 规则

- 始终 L2-n或malise 嵌入s bef或e indexing; FAISS's inner product on n或malised vec到rs equals cosine similarity.
- F或 < 100k 图像s, `IndexFlatIP` (exact) 是simplest 和 fastest.
- F或 100k-10M, `IndexIVFFlat` 是 st和ard trade-的f.
- F或 > 10M, use HNSW 或 一个product-quantised variant.
- 不要 rebuild index on every query; embed once, search many times.
