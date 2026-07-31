Spine/
│
├── v0.3.2 ─ Core Data Layer Stabilization
│   ├── spine/
│   │   ├── data/
│   │   │   ├── __init__.py
│   │   │   ├── dataset.py
│   │   │   ├── dataloader.py
│   │   │   ├── preprocessing.py
│   │   │   ├── augmentation.py
│   │   │   └── transforms.py
│   │   │
│   │   ├── schemas/
│   │   │   ├── patient.py
│   │   │   ├── vertebra.py
│   │   │   ├── study.py
│   │   │   └── annotation.py
│   │   │
│   │   └── io/
│   │       ├── dicom.py
│   │       ├── nifti.py
│   │       └── export.py
│   │
│   └── tests/
│       ├── test_dataset.py
│       ├── test_io.py
│       └── test_schema.py
│
├── v0.3.3 ─ Segmentation Framework
│   ├── spine/
│   │   ├── segmentation/
│   │   │   ├── __init__.py
│   │   │   ├── base.py
│   │   │   ├── vertebra.py
│   │   │   ├── disc.py
│   │   │   ├── spinal_cord.py
│   │   │   ├── metrics.py
│   │   │   └── visualization.py
│   │   │
│   │   └── losses/
│   │       ├── dice.py
│   │       ├── focal.py
│   │       └── hybrid.py
│   │
│   └── tests/
│       ├── test_segmentation.py
│       └── test_losses.py
│
├── v0.3.4 ─ Detection Module
│   ├── spine/
│   │   ├── detection/
│   │   │   ├── fractures.py
│   │   │   ├── scoliosis.py
│   │   │   ├── stenosis.py
│   │   │   ├── tumors.py
│   │   │   ├── degeneration.py
│   │   │   └── anomalies.py
│   │   │
│   │   └── inference/
│   │       ├── pipeline.py
│   │       ├── predictor.py
│   │       └── postprocessing.py
│   │
│   └── tests/
│       ├── test_detection.py
│       └── test_pipeline.py
│
├── v0.3.5 ─ Machine Learning Foundation
│   ├── spine/
│   │   ├── models/
│   │   │   ├── cnn.py
│   │   │   ├── unet.py
│   │   │   ├── resnet.py
│   │   │   ├── transformer.py
│   │   │   └── registry.py
│   │   │
│   │   ├── training/
│   │   │   ├── trainer.py
│   │   │   ├── callbacks.py
│   │   │   ├── optimizer.py
│   │   │   └── scheduler.py
│   │   │
│   │   └── evaluation/
│   │       ├── benchmark.py
│   │       ├── metrics.py
│   │       └── reports.py
│   │
│   └── tests/
│       ├── test_models.py
│       └── test_training.py
│
├── v0.3.6 ─ Clinical Analysis
│   ├── spine/
│   │   ├── measurements/
│   │   │   ├── cobb_angle.py
│   │   │   ├── disc_height.py
│   │   │   ├── vertebral_volume.py
│   │   │   ├── curvature.py
│   │   │   └── alignment.py
│   │   │
│   │   ├── reports/
│   │   │   ├── clinical.py
│   │   │   ├── summary.py
│   │   │   └── templates.py
│   │   │
│   │   └── statistics/
│   │       ├── population.py
│   │       └── analytics.py
│   │
│   └── tests/
│       ├── test_measurements.py
│       └── test_reports.py
│
├── v0.3.7 ─ Visualization Platform
│   ├── spine/
│   │   ├── visualization/
│   │   │   ├── viewer2d.py
│   │   │   ├── viewer3d.py
│   │   │   ├── overlays.py
│   │   │   ├── rendering.py
│   │   │   ├── animation.py
│   │   │   └── export.py
│   │   │
│   │   └── ui/
│   │       ├── dashboard.py
│   │       └── widgets.py
│   │
│   └── tests/
│       └── test_visualization.py
│
├── v0.3.8 ─ Dataset & Benchmark Suite
│   ├── datasets/
│   │   ├── sample_ct/
│   │   ├── sample_mri/
│   │   ├── annotations/
│   │   └── metadata/
│   │
│   ├── benchmarks/
│   │   ├── segmentation/
│   │   ├── detection/
│   │   ├── classification/
│   │   └── performance/
│   │
│   └── docs/
│       └── benchmark_protocol.md
│
├── v0.3.9 ─ Research Toolkit
│   ├── spine/
│   │   ├── experiments/
│   │   │   ├── configs/
│   │   │   ├── ablations.py
│   │   │   ├── reproducibility.py
│   │   │   └── sweeps.py
│   │   │
│   │   └── publication/
│   │       ├── figures.py
│   │       ├── tables.py
│   │       └── supplementary.py
│   │
│   └── docs/
│       └── research_workflow.md
│
└── v0.4.0 ─ First Stable Scientific Platform
    ├── spine/
    │   ├── api/
    │   │   ├── server.py
    │   │   ├── endpoints.py
    │   │   ├── authentication.py
    │   │   └── schemas.py
    │   │
    │   ├── cli/
    │   │   ├── diagnose.py
    │   │   ├── segment.py
    │   │   ├── detect.py
    │   │   ├── benchmark.py
    │   │   └── report.py
    │   │
    │   ├── plugins/
    │   │   ├── registry.py
    │   │   └── interfaces.py
    │   │
    │   └── deployment/
    │       ├── docker.py
    │       ├── cloud.py
    │       └── edge.py
    │
    ├── docs/
    │   ├── API.md
    │   ├── CONTRIBUTING.md
    │   ├── CHANGELOG.md
    │   ├── ROADMAP.md
    │   └── CITATION.cff
    │
    ├── examples/
    │   ├── segmentation_demo.py
    │   ├── detection_demo.py
    │   ├── training_demo.py
    │   └── clinical_report_demo.py
    │
    └── release/
        ├── release_notes.md
        ├── migration_0.3.x_to_0.4.0.md
        └── reproducibility_checklist.md
