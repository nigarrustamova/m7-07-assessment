# Model Lifecycle and Registry Gates

This document defines the lifecycle process of our personalization model, outlining how we train, validate, register, deploy, and monitor models.

## End-to-End Lifecycle Process

The following diagram illustrates the lifecycle stages of a model version:

```mermaid
stateDiagram-v2
    [*] --> Training : Daily scheduled run
    Training --> Registration : Save weights + log metrics
    
    state Registration {
        [*] --> Draft : Logged in MLflow
        Draft --> Automated_Testing : Tagged as 'candidate'
    }
    
    state Validation_Gates {
        Automated_Testing --> Latency_Check : Passes ROC-AUC gate
        Latency_Check --> Security_Scan : CPU inference < 25ms
        Security_Scan --> Promoted_to_Staging : No vulnerabilities
    }
    
    Promoted_to_Staging --> Shadow_Deployment : CD deploys staging instance
    
    state Shadow_Deployment {
        [*] --> Run_Shadow : Receives 100% production traffic replica
        Run_Shadow --> Compare_Metrics : Log predictions & latency
        Compare_Metrics --> Promotion_SignOff : Shadow p95 < 120ms & zero errors
    }
    
    Promotion_SignOff --> Production_A_B : Owner manual approval
    
    state Production_A_B {
        [*] --> Route_5_Percent : A/B Test Router
        Route_5_Percent --> Scale_to_50_Percent : Conversion metrics stable
        Scale_to_50_Percent --> Promoted_to_Active : Wins A/B test
    }
    
    Promoted_to_Active --> Monitoring : Live serving
    Monitoring --> Retrain_Trigger : Drift detected / daily schedule
    Retrain_Trigger --> Training : Restart loop
    
    %% Failures
    Validation_Gates --> Rollback : Fails any gate
    Shadow_Deployment --> Rollback : Latency/Error spike
    Production_A_B --> Rollback : Negative business impact
    Rollback --> [*] : Alert on-call / rollback to v-prev
```

### Lifecycle Phases Details

1. **Training Phase**: The training job runs on a schedule (daily) using the latest 30-day user behavior features from the Lakehouse. It produces a new LightGBM ranker.
2. **Registration Phase**: The model is saved in our Model Registry (MLflow) with complete lineage linking it back to the training code Git commit and the dataset version.
3. **Validation Gates (Automated)**:
   - **Performance Gate**: Checks if the model's offline evaluation metric (e.g., ROC-AUC or MRR) is better than the baseline.
   - **Latency Gate**: Measures prediction latency in a test container environment (must be under 25ms under simulated load).
   - **Security Gate**: Performs dependency checks on the container packaging environment.
4. **Shadow Deployment**: The new candidate model is deployed in production but does *not* return recommendations to real users. Instead, the router duplicates incoming traffic, sends it to the shadow container, logs its performance/latency, and discards the output.
5. **Promotion Sign-Off**: The model owner reviews the shadow metrics. If the model proves stable, they sign off to begin A/B testing.
6. **Production A/B serving**: Traffic is gradually routed to the new model (e.g., starting at 5% and scaling to 50%). If business metrics (such as click-through rate) remain stable or improve, it becomes the new default model.
