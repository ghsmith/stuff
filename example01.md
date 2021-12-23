with qualifying_diagnosis as 
( 
  select 
    ld.diagnosis_key,  
    ld.diagnosis_cd,  
    ld.diagnosis_desc  
  from  
    ehcvw.lkp_diagnosis ld  
  where  
       ld.diagnosis_cd like 'C83.0%' /* similar to Small cell B-cell lymphoma, unspecified site */ 
    or ld.diagnosis_cd like 'C83.3%' /* similar to Diffuse large B-cell lymphoma, unspecified site */ 
    or ld.diagnosis_cd like 'C85.1%' /* similar to Unspecified B-cell lymphoma, unspecified site */ 
    or ld.diagnosis_cd like 'C85.2%' /* similar to Mediastinal (thymic) large B-cell lymphoma, unspecified site */ 
    or ld.diagnosis_cd like 'C88.4'  /* Extranodal marginal zone B-cell lymphoma of mucosa-associated lymphoid tissue [MALT-lymphoma] */ 
    or ld.diagnosis_cd like 'C91.1%' /* similar to Chronic lymphocytic leukemia of B-cell type not having achieved remission */ 
    or ld.diagnosis_cd like 'C91.3%' /* similar to Prolymphocytic leukemia of B-cell type not having achieved remission */ 
    or ld.diagnosis_cd like 'C91.A%' /* similar to Mature B-cell leukemia Burkitt-type not having achieved remission */ 
    or ld.diagnosis_cd like 'C90.%'  /* similar to Multiple myeloma not having achieved remission */ 
    or ld.diagnosis_cd like 'C34.%'  /* similar to Malignant neoplasm of bronchus and lung */ 
), 
sars2_positive_patient_key as 
( 
  select 
    patient_key, 
    max(specimen_collect_dt) keep(dense_rank last order by specimen_collect_dt) latest_spec_coll_dt, 
    max(encounter_key) keep(dense_rank last order by specimen_collect_dt) latest_result_enc_key, 
    ( 
      select max(fe.encounter_key) 
      from   ehcvw.fact_encounter fe 
             /* Millennium patient_id:CDW patient key is 1:many (eg, every phone # update gets a new patient_key) */ 
      where  fe.patient_key in(select patient_key from ehcvw.lkp_patient where patient_id = (select patient_id from ehcvw.lkp_patient where patient_key = frl.patient_key)) 
        and  fe.status_encounter_key = 9 /* Active */ 
        and  fe.type_encounter_key in (17, 18) /* Inpatient, Emergency */ 
    ) 
    active_ip_enc_key 
  from 
    ehcvw.fact_result_lab frl 
  where 
    frl.day_result_verify_key > 20210101 
    and frl.structured_result_type_key in 
    ( 
      select 
        structured_result_type_key 
      from 
        ehcvw.lkp_structured_result_type 
      where 
        structured_result_type_id in 
        ( 
          select 
            to_char(ese.event_cd) event_set_cd 
          from 
            hnamdwh.v500_event_set_explode ese 
          where 
            ese.event_set_cd = 3518196035 /* event set 'Coronavirus CoVID19 by PCR' */ 
        ) 
    ) 
    and frl.result_lab_tval in('Positive', 'Detected', 'PRESUMPTIVE POS') 
  group by 
    patient_key 
) 
select 
  lp.empi_nbr, 
  ( 
    select 
      listagg('[' || to_char(latest_recorded_dt, 'YYYY') || ']' || diagnosis_cd, ';') within group(order by diagnosis_cd) 
    from 
      ( 
        select 
          substr(qd.diagnosis_cd, 1, instr(qd.diagnosis_cd, '.') - 1) diagnosis_cd, 
          min(qd.diagnosis_desc) keep(dense_rank first order by qd.diagnosis_cd) diagnosis_desc, 
          max(fed.recorded_dt) latest_recorded_dt 
        from 
          ehcvw.fact_event_diagnosis fed 
            join qualifying_diagnosis qd on(qd.diagnosis_key = fed.diagnosis_key) 
        where 
          /* Millennium patient_id:CDW patient key is 1:many (eg, every phone # update gets a new patient_key) */ 
          fed.patient_key in(select patient_key from ehcvw.lkp_patient where patient_id = (select patient_id from ehcvw.lkp_patient where patient_key = sppk.patient_key)) 
          and fed.problem_classification_key = 0 /*  Medical */ 
        group by 
          substr(qd.diagnosis_cd, 1, instr(qd.diagnosis_cd, '.') - 1) 
      ) 
  ) 
  qualifying_diag_list_short, 
  ( 
    select 
      listagg('[' || to_char(latest_recorded_dt, 'YYYY') || ']' || diagnosis_cd || '-' || diagnosis_desc, ';' || chr(13) || chr(10)) within group(order by diagnosis_cd) 
    from 
      ( 
        select 
          qd.diagnosis_cd, 
          min(qd.diagnosis_desc) keep(dense_rank first order by qd.diagnosis_cd) diagnosis_desc, 
          max(fed.recorded_dt) latest_recorded_dt 
        from 
          ehcvw.fact_event_diagnosis fed 
            join qualifying_diagnosis qd on(qd.diagnosis_key = fed.diagnosis_key) 
        where 
          /* Millennium patient_id:CDW patient key is 1:many (eg, every phone # update gets a new patient_key) */ 
          fed.patient_key in(select patient_key from ehcvw.lkp_patient where patient_id = (select patient_id from ehcvw.lkp_patient where patient_key = sppk.patient_key)) 
          and fed.problem_classification_key = 0 /*  Medical */ 
        group by 
          qd.diagnosis_cd 
      ) 
  ) 
  qualifying_diag_list_long, 
  sppk.latest_spec_coll_dt latest_sars2_pcr_pos_result, 
  to_char(lp.patient_birth_dt, 'YYYY') year_of_birth, 
  lp.gender_cd, 
  lp.race_cd, 
  fe_active.encounter_nbr active_ip_enc_fin, 
  fe_active.arrival_dt active_ip_enc_arrival_dt, 
  fe_active.discharge_dt active_ip_enc_disch_dt, 
  (select type_encounter_desc from ehcvw.lkp_type_encounter where type_encounter_key = fe_active.type_encounter_key) active_ip_enc_type, 
  (select campus_cd from ehcvw.lkp_location where location_key = fe_active.location_key) active_ip_enc_fac, 
  (select unit_cd from ehcvw.lkp_location where location_key = fe_active.location_key) active_ip_enc_unit, 
  fe_result.encounter_nbr result_enc_fin, 
  fe_result.arrival_dt result_enc_arrival_dt, 
  fe_result.discharge_dt result_enc_disch_dt, 
  (select type_encounter_desc from ehcvw.lkp_type_encounter where type_encounter_key = fe_result.type_encounter_key) result_enc_type, 
  (select campus_cd from ehcvw.lkp_location where location_key = fe_result.location_key) result_enc_fac, 
  (select unit_cd from ehcvw.lkp_location where location_key = fe_result.location_key) result_enc_unit 
from 
  sars2_positive_patient_key sppk 
    join ehcvw.lkp_patient lp on(lp.patient_key = sppk.patient_key) 
    join ehcvw.fact_encounter fe_result on(fe_result.encounter_key = sppk.latest_result_enc_key) 
    left outer join ehcvw.fact_encounter fe_active on(fe_active.encounter_key = sppk.active_ip_enc_key) 
where 
  exists 
  ( 
    select 
      'x' 
    from 
        ehcvw.fact_event_diagnosis fed 
    where 
      /* Millennium patient_id:CDW patient key is 1:many (eg, every phone # update gets a new patient_key) */ 
      fed.patient_key in(select patient_key from ehcvw.lkp_patient where patient_id = (select patient_id from ehcvw.lkp_patient where patient_key = sppk.patient_key)) 
      and fed.problem_classification_key = 0 /*  Medical */ 
      and fed.diagnosis_key in(select diagnosis_key from qualifying_diagnosis) 
  ) 
order by 
  sppk.latest_spec_coll_dt desc, 
  empi_nbr
