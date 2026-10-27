# **Project Name:** Protein Crystallization Imaging Analyzer

# Team Members:

1. Vincent Kenutama Prasetyo  
2. Faundra Pratama Sukma

# Problem (The "Why")

Protein crystallization experiments generate hundreds to thousands of high-resolution droplet images per 96-well plate across multiple inspection days. Researchers must inspect each individual well and classify its state (e.g., Clear, Precipitate, Crystal, or Other) to identify optimal conditions for protein crystal growth.

* **Current Issue:** Crystallization drop scoring is predominantly performed manually. Researchers manually browse through hundreds of images well-by-well and log observations into spreadsheets. This process is time-consuming, prone to human fatigue, and suffers from high inter- and intra-observer variability. Furthermore, there is no integrated workflow to retrain or fine-tune classification models when new experimental imaging conditions are introduced.
* **Impact:** Structural biologists and lab technicians spend hours per plate on repetitive visual inspection, increasing the risk of missing rare, high-value crystal hits or discovering them too late. Inconsistent manual scoring undermines data reproducibility across experimental batches, while the lack of continuous learning prevents automated tools from adapting to specific laboratory imaging setups.

# Proposed Solution

A modern full-stack web application with a **C# .NET 8 Web API** backend and a **React TSX** frontend that unifies image labeling, automated model training, batch plate inference, and a closed-loop active learning review workflow.

* **Our Approach:** The application provides a seamless, closed-loop lifecycle for protein crystallization analysis. Users can annotate droplet images and trigger automated model training from .NET 8, which orchestrates Python-based transfer learning and compiles the trained model into high-performance ONNX format. For inspection, the system ingests plate images, maps them onto a standard 96-well grid (A1 to H12), and executes asynchronous inference via ONNX Runtime to produce predictions with confidence scores. Any uncertain predictions or false classifications can be verified, corrected by the researcher, and automatically fed back into the training dataset for continuous improvement.
* **Key Value:** Transforms hours of tedious manual inspection into an instantaneous, automated first-pass evaluation while keeping the scientist in full control (Human-in-the-Loop). By coupling inference directly with an active learning feedback loop, the model continuously adapts and increases in accuracy. Strict adherence to Clean Architecture ensures that core domain logic, training orchestration, and inference contracts remain decoupled from external libraries, guaranteeing 100% unit-testable business logic.

# Core Features

* **Feature 1: Dataset Annotation & Automated Model Training Pipeline**  
  An interactive web interface for labeling crystallization images into standard categories (*Clear*, *Precipitate*, *Crystal*, *Other*). Once labeled, users can trigger an automated training pipeline: .NET 8 orchestrates a Python training script in the background, streams training progress and metrics in real time, and exports the fine-tuned model into an optimized ONNX artifact ready for immediate deployment.
* **Feature 2: Automated Plate Inference, Grid Mapping & Inspection Reporting**  
  Batch-ingests droplet images and automatically maps them to standard 96-well plate coordinates (A1 through H12). High-throughput asynchronous inference runs via in-process ONNX Runtime in .NET 8, returning class predictions and confidence scores for every well. The full plate state is persisted in SQLite, and researchers can export structured inspection summaries (CSV/tabular format) for laboratory records.
* **Feature 3: Interactive Review & Active Learning Feedback Loop**  
  A color-coded 96-well plate viewer featuring an automated *Confidence Triage* mechanism that highlights low-confidence wells or critical *Crystal* detections for immediate human verification. Researchers can override any misclassified well with a single click. Overridden samples are automatically routed back into the labeled training pool (Feature 1), creating a continuous active learning flywheel that progressively enhances model accuracy.

# High-Level Architecture & Quality Assurance

* **Architecture Pattern:** Clean Architecture (Domain, Application, Infrastructure, Web API) paired with a modular React TSX frontend. The Domain and Application layers are strictly isolated from UI frameworks, external Python processes, and database implementations, ensuring pure, deterministic business logic.
* **Design Patterns Used:**  
  * **Repository Pattern:** For abstracting persistent access to plate configurations, well images, annotations, and inspection histories.  
  * **Dependency Injection:** Centralized IoC wiring across all layers to ensure loose coupling and seamless test mocking.  
  * **Strategy Pattern:** Decoupling image preprocessing pipelines and inference engine implementations behind swappable contracts (`IInferenceEngine`, `IImagePreprocessor`).  
  * **Observer / Event-Driven Pattern:** Asynchronous event notifications for background Python training progress and batch inference completion.  
  * **Factory Pattern:** Thread-safe instantiation of ONNX runtime inference sessions and execution providers.
* **Code Quality Goal:** We commit to maintaining a green GitLab CI/CD pipeline, writing unit tests to be tracked via Cobertura (targeting 100% coverage on core business logic), and keeping Qodana static analysis warnings below 10 (Target: 0).

# Identified Challenges & Risks

* **Risk 1: Reliable Process Orchestration & Real-Time Log Streaming (.NET ↔ Python)**  
  Managing asynchronous background Python training processes from .NET 8 requires robust process lifecycle handling. Unhandled process crashes, deadlocks in standard I/O streams, or orphaned processes upon application shutdown can degrade system stability.  
  *Mitigation:* Encapsulate external process execution inside a dedicated infrastructure service that implements strict process supervision, standard output/error stream redirection with asynchronous line buffering, and graceful termination via cancellation tokens and OS process tree cleanup.
* **Risk 2: Thread-Safe Batch Inference & Native Memory Management**  
  Evaluating 96 or more high-resolution well images concurrently can introduce memory spikes and native heap fragmentation due to unmanaged buffers in OpenCV and ONNX Runtime.  
  *Mitigation:* Isolate native operations behind strict `IDisposable` scopes, utilize semaphore-throttled batch processing to cap peak memory footprint, and maintain a shared, read-only ONNX `InferenceSession` for thread-safe concurrent evaluations.
* **Risk 3: Class Imbalance & Feedback Quality in the Active Learning Loop**  
  Protein crystal drops are statistically rare compared to clear drops or amorphous precipitates. Naive retraining on newly overridden data could cause model degradation or bias toward dominant classes.  
  *Mitigation:* Implement class-weighted loss and stratified dataset balancing during the fine-tuning phase, accompanied by regression validation against a held-out benchmark set before activating the retrained model.
