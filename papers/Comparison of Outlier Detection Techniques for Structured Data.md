---
link: https://arxiv.org/pdf/2106.08779
---

## Proximity-Based Methods (looking at how close/far points are from others):

1. Local Outlier Factor (LOF)
- Checks how isolated each data point is compared to its neighbors
- Gives each point a score based on how different its density is from its neighbors
- Higher scores mean the point is more likely to be an outlier

2. K Nearest Neighbors (KNN)
- Looks at the distance to a point's k closest neighbors
- If a point is very far from its nearest neighbors, it's likely an outlier
- Uses a threshold to decide what "too far" means

3. Clustering-Based Local Outlier Factor (CBLOF)
- First groups similar data points into clusters
- Then uses LOF principles to find outliers
- Considers both cluster size and distance when deciding if something is an outlier

4. Angle-Based Outlier Detection (ABOD)
- Instead of just looking at distances, looks at angles between points
- Points deep inside a cluster have high angle variance
- Outliers tend to have small angle variance because they're far away

5. Histogram-Based Outlier Score (HBOS)
- Creates histograms for each feature in the data
- Points in low-density areas of the histogram are likely outliers
- Fast but assumes features are independent

## Linear Model Based Methods:

6. Minimum Covariance Determinant (MCD)
- Creates a robust estimate of data center and spread
- Less sensitive to extreme values than regular statistical measures
- Good for detecting outliers in multiple dimensions

7. One-Class Support Vector Machines (OCSVM)
- Learns the boundary of normal data
- Anything outside this boundary is considered an outlier
- Good for complex data patterns

8. Principal Component Analysis (PCA)
- Reduces data to its main components
- Points that don't fit well with these components are outliers
- Good for high-dimensional data

## Ensemble Methods (combining multiple approaches):

9. Feature Bagging
- Runs multiple outlier detection methods on different subsets of features
- Combines the results for better accuracy
- Good for noisy, high-dimensional data

10. Isolation Forest
- Randomly divides data into smaller groups
- Outliers are isolated in fewer divisions
- Fast and effective for large datasets

11. LODA (Lightweight Online Detector of Anomalies)
- Creates multiple simple detectors
- Combines their results
- Good for real-time detection

12. LSCP (Locally Selective Combination in Parallel)
- Combines multiple detection methods
- Focuses on local regions around each point
- Adapts to different parts of the data

## Probabilistic Method:

13. COPOD (Copula-Based Outlier Detection)
- Uses probability theory to model relationships between features
- Calculates how extreme each point is
- Good for understanding complex feature relationships

14. COF (Connectivity-Based Outlier Factor)
- Similar to LOF but considers paths between points
- Looks at how well connected points are
- Good for detecting outliers in complex patterns

15. SOD (Subspace Outlier Detection)
- Looks for outliers in different subsets of features
- Good for data where outliers only show up in some dimensions
- Helps when not all features are relevant for outlier detection

