# Anomaly Detection in NASA Spacecraft Sensor Data
<img src="https://github.com/StarRider/Anomaly-Detection-in-NASA-Spacecraft-s-Time-Series-Data/assets/30108439/6f87ede5-f5ba-4c1f-a877-32788fcb1f7f" width="600">

Image by [NASA](https://ai.jpl.nasa.gov/)

## Objective
To develop a system that can detect any unusual activity in the data collected from NASA satellites and spacecraft. The system is designed to identify any anomalies and notify technicians, thereby reducing the time and effort required to manually review all the data. My aim is to minimize the number of false detections (both positive and negative) in order to achieve maximum accuracy and to improve the architecture's present performance.

## Project Scenario
NASA has developed an innovative approach called dynamic thresholding to detect anomalies in time series data using LSTM models. I have fine-tuned and improved the model's performance to achieve greater accuracy and minimize false detections using anomaly pruning. This system reduces the monitoring burden on operations engineers and minimizes operational risk, bringing us closer to unlocking the mysteries of the universe.

## Anomaly in Time Series
### What is an anomaly in time series data?
With regards to time series, any unusual value that seems to be out of normal is an anomaly. There are three types of anomalies with respect to time series as per the NASA engineers,
#### Point Anomalies:

* Point anomalies refer to individual data points that are considered abnormal when compared to the rest of the data.
* These anomalies are characterized by their deviation from the normal behavior of the time series.
* They typically occur as isolated instances within the time series and are identifiable as extreme values or outliers.
* Point anomalies can be caused by various factors such as measurement errors, sensor malfunctions, or rare events.
#### Collective Anomalies:

* Collective anomalies occur when a sequence of data points exhibits abnormal behavior as a whole, rather than any single point being anomalous by itself.
* Instead of isolated extreme values, collective anomalies are identified by patterns or trends in the time series data that deviate significantly from the expected or normal behavior.
* These anomalies may arise due to systemic issues, unexpected changes in underlying processes, or complex interactions among multiple variables over time.
#### Contextual Anomalies:

* Contextual anomalies are single data points that may not appear abnormal when viewed in isolation but are considered anomalous with respect to the local context or neighboring data points.
* Unlike point anomalies, contextual anomalies are identified by comparing a data point to its surrounding context or historical pattern within a specific time window.
* These anomalies often require contextual information or domain knowledge to distinguish them from normal variations in the data.
* Contextual anomalies can be caused by subtle changes in underlying conditions, shifts in behavior over time, or contextual dependencies within the time series.

Note: This project has classified anomalies into two categories, point and contextual. (Here, we have clubbed the collective and contextual anomalies together, calling it contextual anomalies)

## Anomaly Detection System Architecture
![image](https://github.com/StarRider/Anomaly-Detection-in-NASA-Spacecraft-s-Time-Series-Data/assets/30108439/33474489-c533-4e0d-8a3c-c24aa30ea403)

## Dataset - Overview
We have the anonymized data from two satellites, Soil Moisture Active Passive satellite (SMAP) and the Curiosity Rover on Mars (MSL). The data is placed in `./data/train` and `./data/test`. (You can find this open sourced [data](https://s3-us-west-2.amazonaws.com/telemanom/data.zip) here as well.) The filename should be a unique channel name or ID (like A-1.npy, G-6.npy etc.). The telemetry values being predicted in the test data must be the first feature in the input.

For example, a channel T-1 should have train/test sets named T-1.npy with shapes akin to (4900,61) and (3925, 61), where the number of input dimensions are matching (61). The actual telemetry values should be along the first dimension (4900,1) and (3925,1).

![image](https://github.com/StarRider/Anomaly-Detection-in-NASA-Spacecraft-s-Time-Series-Data/assets/30108439/c834f269-d075-4ef5-a8b3-28c83fecfb10)

We have a labelled dataset for checking the accuracy of our anomaly detection, that file is `labeled_anomalies.csv` which includes:

- `channel id`: anonymized channel id - first letter represents nature of channel (P = power, R = radiation, etc.)
- `spacecraft`: spacecraft that generated telemetry stream
- `anomaly_sequences`: start and end indices of true anomalies in stream
- `class`: the class of anomaly
- `num values`: number of telemetry values in each stream

## Model Architecture - LSTM
The inherent properties of LSTMs makes them an ideal candidate for anomaly detection tasks involving time-series, non-linear numeric streams of data. LSTMs are capable of learning the relationship between past data values and current data values and representing that relationship in the form of learned weights. These advantages have motivated the use of LSTM networks in several recent anomaly detection tasks related papers.

For our purpose we have also chosen LSTM model, for which you can see the high level view below,

![image](https://github.com/StarRider/Anomaly-Detection-in-NASA-Spacecraft-s-Time-Series-Data/assets/30108439/0bc854db-179c-4952-809d-8f6a0e7eae03)

To understand the structure of the model, it is important to understand the shape of input data. Let's suppose we are creating a model a for channel A-1, once you process the data for a channel A-1, your training data will have the following shape:

`X_train shape`: (2620, 250, 25)  The 2nd dimension (the time steps) of the X_train is going to be same for every channel. The first and third dimension is just the number of samples and features that can vary with channels.

`y_train shape`: (2620, 10, 1) The 2nd dimension (10 steps into the future) is the label that the model has to predict. The third dimension is the feature, called telemetry signal value that is what is being predicted.  

(In nutshell, we are using all the features from 250 timesteps in history to train the model to predict one feature upto 10 time steps ahead)

This table below are the parameters for channel A-1 signal. The only parameter that changes w.r.t to a channel is the LSTM input layer size, it is basically put as `(None, X_train.shape[2])`.

| Model Parameter         | Values   |
|-------------------------|----------|
| LSTM1 Input Layer Size  | (250, 25)|
| LSTM1 Hidden Layer Size | 80       |
| Dropout Layer           | 0.3      |
| LSTM2 Hidden Layer Size | 40       |
| Dropout Layer           | 0.3      |
| Dense Output Layer Size | 10       |
| Activation Function     | linear   |
| Loss Metric             | mse      |
| Optimizer               | adam     |

## Dynamic Thresholding
It is a method for analyzing the residuals to compute an appropriate lower and upper threshold to flag the anomalous values and sequences.

Steps for Dynamic Threshold Calculation (Taken from paper):
1. Compute residuals $e_{s}=| y_{t} - \hat{y}_{t}|$
2. Smooth out the residuals using exponentially weighted moving average.
3. Calculate $\mu(e_{s})$ and $\sigma(e_{s})$
4. Compute $\epsilon=\mu(e_{s}) + z*\sigma(e_{s})$ for $z$ value from two to ten (This range is empirical).
5. Compute the best threshold using below equation,

   $\epsilon = \text{argmax}(\epsilon) = \large{\frac{\Delta\mu(e_s)/\mu(e_s) + \Delta\sigma(e_s)/\sigma(e_s)} {\left( |e_{a}| + |E_{seq}|^2 \right)}}$

   $\text{Such that:}$

   $\Delta\mu(e_s) = \mu(e_s) - \mu(\{e_{s} < \epsilon \})$

   $\Delta\sigma(e_s) = \sigma(e_s) - \sigma(\{e_s < \epsilon \})$

   $e_{a} = \{e_s > \epsilon \}$

   $E_{seq} = \text{continuous sequences of } e_{a} 's$

## Anomaly pruning
Once a dynamic threhsold is found we are capable of flagging anomalies. However, large number of false positives can be trouble. To reduce the number of false positives we use anomaly pruning where we reclassify certain anomalous sequences as nominal based on a criteria described below,
1. Create a new set $e_{max} = {max(E_{seq} \text{ from the set of }E_{seq}'s)}$
2. Add to this set the $max(\text{non anomalous errors})$
3. Sort this set in descending order.
4. Step through this set of residuals and compute percentage difference from the previous residual.
5. If the % difference at position $i$ is greater than $p$ (a value that's empirical to NASA) then all the residuals before $i$ would remain anomalous.
6. If this violation of $p$ doesn't happen after position $i$ then those residuals after position $i$ would be re-classified as nominal.

## Evaluation of model
The model is evaluated using $F_{0.5}-score$. This metric puts more weightage on precision and less weightage on recall. The model in repo is tuned to achieve `precision=0.86` and `recall=0.81`

$F_{0.5} = \large{\frac{((1 + \beta^2) * Precision * Recall)}{(\beta^2 * Precision + Recall)}}$

With this equation, we get an $F_{0.5}=0.849$ that is higher than the scores in NASA's paper.

## References
1. Detecting Spacecraft Anomalies Using LSTMs and Nonparametric Dynamic Thresholding - [Paper](https://arxiv.org/pdf/1802.04431.pdf)
2. Machine Learning for Time Series Anomaly Detection - [Report](https://dspace.mit.edu/bitstream/handle/1721.1/123129/1128282917-MIT.pdf?sequence=1&isAllowed=y)
