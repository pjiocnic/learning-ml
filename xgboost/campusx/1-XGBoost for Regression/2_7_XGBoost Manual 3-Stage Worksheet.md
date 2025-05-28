Here is your **"XGBoost Manual 3-Stage Worksheet"** using 3 features: `CGPA`, `Marks_12th`, and `Internship`. This step-by-step example simulates how a 3-stage gradient boosting model (like XGBoost) builds up predictions by adding small trees to correct the previous stage's residuals.

### Key Concepts Illustrated:

1. **Stage 1**:
   Predicts the **mean of the target** (Package) as the initial guess.
2. **Stage 2**:
   Trains a decision stump (tree1) based on `CGPA`, then updates prediction using learning rate `η = 0.3`.
3. **Stage 3**:
   Trains another tree (tree2) based on `Marks_12th` to further reduce the residuals.

### How values were derived:
- **Stage1_Pred**: Mean of all `Package` values.
- **Residual_1**: Package - Stage1_Pred
- **Tree1_Output**: Based on condition `CGPA <= 7.0 → -1.5`, else `+1.2`.
- **Stage2_Pred**: `Stage1_Pred + η × Tree1_Output`
- **Residual_2**: Package - Stage2_Pred
- **Tree2_Output**: Based on condition `Marks_12th <= 78 → +0.8`, else `-0.5`
- **Stage3_Pred**: `Stage2_Pred + η × Tree2_Output`
- **Residual_3**: Package - Stage3_Pred

Would you like the same logic extended into a downloadable worksheet or visual decision trees drawn from this?