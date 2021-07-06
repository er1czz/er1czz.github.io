# Synthesizing health insurance claim payload

## Background 
- Building a predictive model to detect claim denial
    - Input: 837 info
    - Label: reason code from 835
    - Output: denial or not and reason codes

 Health insurance claim payload is in 837 EDI format (typically JSON in production). 
 The training data, however, are typically long-term stored in relational databases.
 Very often, it is necessary to convert training data into payload format for the test purposes. 

## Payload
- Payload:


```
# function of interest 
# >>> synth() <<<
# default database: "clai"."medical_network_extract_v36"  
# input format: "EP020421796423531", 'EP020821709092846'
# If user != Eric, please update your s3_dir below

# 1. fetch data as dataframe through Athena
def sampler(claim_tcn_id='EP041721702869330'):
    import pandas as pd
    from pyathena import connect
    from pyathena.pandas.util import as_pandas
    
    claim_tcn_id = str(claim_tcn_id)

    s3_dir = 's3://nucleus-chc-preprod-datasciences/users/erzhang/athena-queries/'
    region = 'us-east-1'
    cursor = connect(s3_staging_dir= s3_dir, region_name= region).cursor()
    
    query = """
    SELECT  claim_tcn_id, service_line_number, 
    denial_code, payer_id, payer_name, 
    claim_filing_indicator_cd, 
    service_line_charge_amount,
    rendering_prov_npi,
    rendering_prov_taxonomy,
    billing_prov_npi,
    procedure_code,
    procedure_modifier_1, procedure_modifier_2, procedure_modifier_3, procedure_modifier_4,
    principal_diagnosis_code, claim_received_date, service_to_date, patient_state
    FROM "clai"."medical_network_extract_v36"  
    WHERE claim_tcn_id IN ('EP041721702869330')
    ORDER BY claim_tcn_id, CAST(service_line_number AS INTEGER)
    ;  """
    
    query = query.replace('EP041721702869330',claim_tcn_id)
    query = query.replace("('('","('")
    query = query.replace("')')","')")

    cursor.execute(query)
    sample = as_pandas(cursor)
    
    return sample

# 2. conversion main
def payload_gen(synth):  
    import pandas as pd
    import numpy as np
    
    claims = pd.unique(synth.claim_tcn_id)
    
    output = str(' ')
    
    for i in np.arange(len(claims)):
        claim_tcn_id = claims[i]
        cid = claim_tcn_id
        # creating modifiers
        synth = synth.replace(r'^\s*$', np.nan, regex=True) # replace blank with null
        cols = ['procedure_code', 'procedure_modifier_1', 'procedure_modifier_2', 'procedure_modifier_3', 'procedure_modifier_4']
        tmp = synth[cols]
        tmp = tmp.replace(r'^\s*$', np.nan, regex=True)
        synth['procedure_modifiers'] = \
            tmp[tmp.columns[1:]].apply(lambda x: ','.join(x.dropna().astype(str)), axis=1)
        synth['procedure_modifiers'] = synth['procedure_modifiers'].str.split(',')

        # single value - 10 fields
        
        payer_id = \
                pd.unique(synth[synth.claim_tcn_id==cid].payer_id).item()
        payer_name = \
                pd.unique(synth[synth.claim_tcn_id==cid].payer_name).item()
        claim_filing_indicator_cd = pd.unique(synth[synth.claim_tcn_id==cid].claim_filing_indicator_cd).item()
#        subscriber_id = pd.unique(synth[synth.claim_tcn_id==cid].subscriber_id).item()
        rendering_prov_npi = \
                pd.unique(synth[synth.claim_tcn_id==cid].rendering_prov_npi).item()
        rendering_prov_taxonomy = \
                pd.unique(synth[synth.claim_tcn_id==cid].rendering_prov_taxonomy).item()
        billing_prov_npi = \
                pd.unique(synth[synth.claim_tcn_id==cid].billing_prov_npi).item()
        principal_diagnosis_code = \
                pd.unique(synth[synth.claim_tcn_id==cid].principal_diagnosis_code).item()
        claim_received_date = \
                pd.unique(synth[synth.claim_tcn_id==cid].claim_received_date).item()
        patient_state = \
                pd.unique(synth[synth.claim_tcn_id==cid].patient_state).item()        
        
        # list of values - 5 fields

        service_line_number = \
                synth[synth.claim_tcn_id==cid].service_line_number.values.tolist()
        service_line_charge_amount = \
                synth[synth.claim_tcn_id==cid].service_line_charge_amount.values.tolist()
        procedure_code = \
                synth[synth.claim_tcn_id==cid].procedure_code.values.tolist()
        procedure_modifiers = \
                synth[synth.claim_tcn_id==cid].procedure_modifiers.values.tolist()
        service_to_date = \
                synth[synth.claim_tcn_id==cid].service_to_date.values.tolist()
        denial_code = \
                synth[synth.claim_tcn_id==cid].denial_code.values.tolist()
        
        payload = {'claim_tcn_id': cid, \
                   'payer_id': payer_id, \
                   'payer_name': payer_name, \
                   'claim_filing_indicator_cd': claim_filing_indicator_cd, \
#                   'subscriber_id': subscriber_id, \
                   'rendering_prov_npi': rendering_prov_npi, \
                   'rendering_prov_taxonomy': rendering_prov_taxonomy, \
                   'billing_prov_npi': billing_prov_npi, \
                   'principal_diagnosis_code': principal_diagnosis_code, \
                   'claim_received_date': claim_received_date, \
                   'patient_state': patient_state, \
                   'service_line_number': service_line_number, \
                   'service_line_charge_amount': service_line_charge_amount, \
                   'procedure_code': procedure_code, \
                   'procedure_modifiers': procedure_modifiers, \
                   'service_to_date': service_to_date, \
                   'denial_code': denial_code
                  }
            
        text = f'payload_{i+1} = {payload}'
        text = str(text).replace("''","")
        text = str(text).replace("nan","[]")
        text = str(text).replace("'", '"')
        output = output + "\n \n" + text
    return output

# 3. output

# 3.1 print out as native medical network format
def payload_mn(data):
    output = '### Native Payload for Medical Network ### \n' 
    output = output + str(payload_gen(data))
    print(output)
    
# 3.2 print out as Lambda compatible format
def payload_test(data):
    output = '### Payload for Lambda Testing ### \n' 
    output = output + str(payload_gen(data))
    output = str(output).replace('"', r'\"')
    print(output)
    
# finalization
def synth(claim_tcn_id):
    sample = sampler(claim_tcn_id)
    
# output as native medical network format
#    payload_mn(sample)  

# output as lambda-friendly format 
    payload_test(sample)
```
