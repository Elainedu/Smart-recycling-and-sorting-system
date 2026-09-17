# Smart-recycling-and-sorting-system

**Streamlit + fastai ResNet-based waste image classifier, with two additional Hugging Face baselines for comparison.**

An end-to-end waste-classification demo built for the *2024 Summer New Talent
(新尖兵)* programme. The core is the community-authored
[Rootstrap fastai waste classifier](https://github.com/Rootstrap/fastai-waste-classifier)
(vendored under `fastai-waste-classifier-main/`) which trains a ResNet on a
7-class recyclables dataset and serves the model through a Streamlit web
front-end. Two extra notebooks at the repository root reuse pre-trained
Hugging Face image-classification models as drop-in baselines against the same
7-class problem.

---

## Model & dataset

### Classes (7)

The classifier predicts one of:

- `cardboard`
- `compost`
- `glass`
- `metal`
- `paper`
- `plastic`
- `trash`

### Dataset

The training / test split originates from Bing Image Search (the fetching
code lives in a linked Colab in the vendored `fastai-waste-classifier-main/README.md`).
The final split archive is hosted on Google Drive; each `train/` and `test/`
directory contains one sub-folder per class. See
`fastai-waste-classifier-main/README.md` for the download link.

### Model

- **Primary:** `fastai` `vision_learner` fine-tuned on top of a
  torchvision ResNet backbone (ResNet-50 for the confusion matrix reported
  in the vendored README, `~0.98` validation accuracy). The serving code
  loads a smaller **ResNet-34** artefact `result-resnet34.pkl` for a lower
  memory footprint. A `result-resnet50.pkl` checkpoint is also included.
- **Baseline #1:** `rootstrap-org/waste-classifier` on Hugging Face
  (`AutoModelForImageClassification`), invoked in
  `RootstrapOrgwasteClassifier.ipynb`.
- **Baseline #2:** `yangy50/garbage-classification` on Hugging Face,
  invoked in `Yangy50GarbageClassification.ipynb`.

---

## How it works

```
             +----------------+
             | user uploads   |
             | an image (jpg, |
             | png)           |
             +--------+-------+
                      |
                      v
       +---------------------------------+
       | Streamlit app  (docker/main.py) |
       |                                 |
       |  1. save upload to timestamped  |
       |     tmp file                    |
       |  2. fastai Learner.predict(...) |
       |     -> (class, class_id, probs) |
       |  3. render class + probability  |
       |     in the right column         |
       |  4. delete tmp file             |
       +----------------+----------------+
                        |
                        v
              +-------------------+
              | result-resnet34.  |
              | pkl (fastai       |
              | Learner pickle)   |
              +-------------------+
```

### Training pipeline (vendored fastai notebook)

`fastai-waste-classifier-main/resnet-model.ipynb` performs the full
transfer-learning flow:

1. Load the 7-class dataset via a `fastai` `DataLoaders`.
2. Apply standard image augmentation (rotation, flip, contrast).
3. Fine-tune a ResNet backbone with `vision_learner(..., metrics=accuracy)`.
4. Evaluate with `ClassificationInterpretation` and export the confusion
   matrix (`classification_matrix_resnet50.png`).
5. Export to `.pkl` with `learn.export()` for downstream serving.

`fastai-waste-classifier-main/utils.py` provides helper evaluation utilities
used by the notebook.

---

## Repository layout

```
Smart-recycling-and-sorting-system/
├── README.md                                       # this file
├── RootstrapOrgwasteClassifier.ipynb               # HF baseline #1
├── Yangy50GarbageClassification.ipynb              # HF baseline #2
└── fastai-waste-classifier-main/                   # vendored primary project
    ├── README.md                                   # upstream README
    ├── requirements.txt
    ├── resnet-model.ipynb                          # training notebook
    ├── utils.py                                    # eval helpers
    ├── result-resnet34.pkl                         # serving checkpoint (small)
    ├── result-resnet50.pkl                         # 98% acc checkpoint
    ├── classification_matrix_resnet50.png          # confusion matrix
    ├── aws.txt                                     # EC2 deploy notes
    ├── docker/
    │   ├── Dockerfile                              # Streamlit + fastai image
    │   ├── main.py                                 # Streamlit app entry point
    │   └── result-resnet34.pkl                     # bundled model for the image
    ├── static/images/default.png
    ├── templates/upload.html                       # legacy Flask template
    └── test-photos/                                # 14 sample inference images
        ├── basura-envase-plastico.jpeg
        ├── basura-vaso-plastico.jpeg
        ├── botella-plastico.jpeg
        ├── cascara-banana{,2,3}.jpeg
        ├── envase-plastico{,2}.jpeg
        ├── lata{,2}.jpeg
        ├── manzana-mordida.jpeg
        ├── papel-sucio.jpeg
        ├── sobre-de-te.jpeg
        └── tapa-plastico.jpeg
```

---

## Running

### Option A — Streamlit demo (recommended)

```bash
git clone https://github.com/Elainedu/Smart-recycling-and-sorting-system.git
cd Smart-recycling-and-sorting-system/fastai-waste-classifier-main

# Python 3.9+ recommended
pip install -r requirements.txt
# requirements.txt: jupyter notebook fastai glob2 sklearn pandas numpy seaborn flask werkzeug streamlit

streamlit run docker/main.py
```

Open the printed local URL (usually `http://localhost:8501`) and upload any
image from `test-photos/` to try it.

### Option B — Docker

```bash
cd fastai-waste-classifier-main
docker buildx build --platform linux/amd64 -t waste-classifier -f docker/Dockerfile .
docker run -p 8501:8501 waste-classifier
```

### Option C — Retrain

```bash
cd fastai-waste-classifier-main
jupyter notebook resnet-model.ipynb
# download the dataset from the Google Drive link in the vendored README,
# unzip to ./dataset/{train,test}/<class_name>/, then run all cells
```

### Option D — Hugging Face baselines

Open either notebook at the repository root:

```bash
pip install transformers torch pillow requests
jupyter notebook RootstrapOrgwasteClassifier.ipynb        # rootstrap-org/waste-classifier
jupyter notebook Yangy50GarbageClassification.ipynb       # yangy50/garbage-classification
```

Both notebooks download the model on first run and predict on a hard-coded
sample image URL — swap the URL / feed a local `PIL.Image` to test your own
inputs.

---

## Notes on the vendored project

`fastai-waste-classifier-main/` is a snapshot of
[Rootstrap/fastai-waste-classifier](https://github.com/Rootstrap/fastai-waste-classifier)
(MIT-licensed). The primary local additions in this repository are:

- The two Hugging Face baseline notebooks at the root.
- Packaging the whole thing as the *Smart Recycling and Sorting System*
  deliverable for the 2024 Summer New Talent programme.

Upstream credits: Rootstrap for the training notebook, model exports, sample
photos, and Streamlit / Docker serving stack.

---

## License

Repository is MIT (matching the upstream fastai project). Dataset images
scraped via Bing are subject to their respective owners' rights and are
intended for research / educational use only.
