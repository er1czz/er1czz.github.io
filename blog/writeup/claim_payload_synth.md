# Synthesizing health insurance claim payload

## Background 
- Building a predictive model to detect claim denial
    - Input: 837 info
    - Label: reason code from 835
    - Output: denial or not and reason codes

 Health insurance claim payload is in 837 EDI format (typically JSON in production). 
 The training data, however, are typically long-term stored in relational databases.
 Very often, it is necessary to convert training data into payload format for the test purposes. 


