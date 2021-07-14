Conditional Probability, a probabilistic approach to assess the relationship between denial reasons. 

By Eric Zhang (March 21, 2021)

1.	Introduction

•	A claim/service can be denied by multiple reasons. For example, there is a service line that has a procedure that is inconsistent with the modifier(s). Moreover, the claim filing time is exceeding the maximum limit.  Apparently, this service would trigger both denial reasons: CO4 and CO29, respectively. 
•	There are a number of denial reasons that commonly occur together. If those claims with CO4 have high chances to trigger CO29, this may indicate the presence of CO4 is strongly correlated with CO29. Measuring the probability of denial reason CO29 when CO4 occurs can indicate the extent of correlation between CO29 and CO4. 
•	The probability applied here is called conditional probability. According to Wikipedia, “conditional probability is a measure of the probability of an event occurring, given that another event (by assumption, presumption, assertion or evidence) has already occurred”. The main purpose of this study to provide a probabilistic approach to probe the correlation between denial reasons.
 
2.	Method and data

•	Dataset: 502,488,339 professional line records from 2019-01 to 2021-04 ("clai"."medical_network_extract_v35").

•	Denial codes: 21 common denial codes by frequency were selected, including CO97, CO96, CO16, OA18, CO197, CO59, CO234, CO204, OA96, CO252, CO29, CO109, CO18, PI97, CO4, CO22, CO185, COA1, OAB11, COB7, and OAA1. In total, there are 231 unique pairs of denial reasons.

•	Workflow: 
I.	A look-up table was produced through AWS Athena. This table summarizes the number of claims that share the same denial patterns (see Figure 1.). 
II.	The conditional probability between different codes were calculated based on the counts. For example, there are 1,301,266 lines with CO4, 2,139,527 lines with CO29, and 250 lines with both CO29 and CO4. Therefore, the condition probability of CO29 when CO4 was triggered, P(CO29/CO4), is 0.00019 by 250/1,301,266. Likewise, P(CO4/CO29) is 0.00012. 

3.	Outcome
•	It seems that the highest conditional probability among these popular denial codes is 0.029 for (CO234/OA96). This indicates weak correlations among the 21 denial codes in this study.
•	55 pairs of denial reasons have zero probability, such as PI97/CO4. This finding implies that when a claim triggers CO4, there is zero chance that PI97 would occur. These pairs of reason codes with zero conditional probability could serve as a blacklist to improve predictability of claim denial.

