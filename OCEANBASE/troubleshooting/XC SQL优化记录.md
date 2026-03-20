# XC SQL优化记录

## 20251211

### SQL 1

```
Sql_id:0EFBC54950AA54818DA518C6B69F3674
SELECT * FROM AC08 WHERE AAZ648 IN  (SELECT ID FROM V_AC43 WHERE AAC001 = ? AND AAE003 = ?      AND AAE793 LIKE 'ZS%'      AND AAE793 NOT IN ('ZS02', 'ZS12', 'ZS03', 'ZS13', 'ZS06', 'ZS16', 'ZST3'))

AC08已按照aac001 分区，该sql需添加aac001在AC08的查询上
```

##SQL 2

```
sqlid:968A7024CF6E3DF5E4CA203CACC3D345
UPDATE /*+ parallel(4) */ IMP_BASICINFO.azB1 a    SET a.BAA543 ='5'  WHERE exists (select 1           from IMP_BASICINFO.pub_get_money_imp d          where a.aaz288 = TO_NUMBER(d.aaz288)            and A.AAE140 = (SELECT f.BAA902                              FROM IMP_PARAMETER.AAA3 f                             WHERE f.AAA100 = 'XZ_ID'                               AND f.AAA102 = d.AAE140)            and a.aae041 = d.fkssq and d.jzbz = '1') and a.baa543 = '4';


条件a.aae140= ()改为a.aae140 in()
实测发现，此处影响ob执行计划生成
```

## 20251212

### sql1

```SQL
XC历史事项已办查询:
SELECT
    *
FROM
    (
        SELECT
            m.case_id,
            f.flow_id           task_id,
            f.step_code         step_code,
            d.affair_code,
            d.affair_name,
            (
                CASE
                    WHEN nvl('2', '1') = '1' THEN
                        nvl(f.step_name, m.step_name_cur)
                    ELSE
                        m.step_name_cur
                END
            ) step_name,
            m.step_name_cur,
            m.accept_time,
            m.applicant_id,
            m.applicant_type,
            m.applicant_cert,
            m.time_limit_type,
            m.time_limit,
            (
                CASE
                    WHEN nvl('2', '1') = '1'
                         AND m.accept_type = '2'
                         AND m.applicant_name <> m.organ_name THEN
                        m.applicant_name
                        || ' - '
                        || m.organ_name
                    ELSE
                        m.applicant_name
                END
            ) applicant_name,
            m.accept_time + nvl(m.time_limit, 0) promise_date,
            m.acceptor_name,
            f.step_start_time   start_time,
            f.step_end_time     end_time,
            f.step_result,
            m.last_operator_time,
            '' due_day,
            nvl(m.accept_type, '1') accept_type,
            (
                CASE
                    WHEN f.operator_id IS NULL
                         OR f.operator_id = '' THEN
                        '0'
                    ELSE
                        '1'
                END
            ) assigned,
            f.operator_name     assignee,
            m.street_name,
            '0' f_cancel,
            substr(d.run_flag, 4, 1) nav_guide,
            m.extend_col
        FROM
            affair_case        m,
            affair_case_step   f,
            affair_define      d
        WHERE
            m.affair_code = d.affair_code
            AND m.case_id = f.case_id
            AND f.step_end_time >= TO_DATE('2025-11-11', 'yyyy-MM-dd')
            AND m.end_type <> '0'
            AND f.operator_id IN (
                SELECT
                cb.user_id
                FROM
                    cas_user   ca,
                    cas_user   cb
                WHERE
                    ca.shbxdjm = cb.shbxdjm
                    AND ca.user_id = f.operator_id
            )
            AND ( m.organ_id = '202112212003071'
                  OR ( m.applicant_type = '2'
                       AND m.applicant_id = '202112212003071' ) )
            AND ( m.applicant_name LIKE '%胡伟%'
                  OR d.affair_name LIKE '%胡伟%'
                  OR m.street_name LIKE '%胡伟%' )
        ORDER BY
            f.step_end_time DESC,
            f.step_seq DESC
    ) r_
WHERE
    ROWNUM <= 20


问题点:
cas_user走全表，导致查询响应时间过长
且疑似此处逻辑冗余
优化:
开发反馈该sql业务逻辑存在问题，已在修改中

```

### sql2

```
劳动能力鉴定相关sql:
SELECT
    COUNT(1)
FROM
    (
        SELECT
            s.xm        AS xm,
            j.jdlb_id   AS jdlbid,
            s.zjlx      AS zjlx,
            s.zjhm      AS zjhm,
            TO_CHAR(j.sqrq, 'yyyy-mm-dd') AS sqrq,
            (
                SELECT
                    listagg(b.text, ',')
                FROM
                    sbdev.affair_dic_data b
                WHERE
                    b.code = 'LAA_YGJDKB'
                    AND k.jdkb_id = b.value (+)
            ) AS jdkbid,
            j.jdbh      AS jdbh,
            (
                SELECT
                    listagg(b.text, ',')
                FROM
                    sbdev.affair_dic_data b
                WHERE
                    b.code = 'LAA_CX_YGJDJL'
                    AND d.jb = b.value (+)
            ) AS ybjb,
            DECODE(j.jdzt, '20', '已归档', '未归档') AS sfgd,
            j.jd_id     AS jdid,
            DECODE(j.cxyj, '01', '受理', '02', '不受理', '03', '补正材料', '04', '视作放弃') AS cxyj
        FROM
            laa_jbxx     j
            JOIN laa_ygsq     s ON j.jd_id = s.jd_id
            LEFT JOIN laa_ygjdjl   d ON d.jd_id = j.jd_id
            LEFT JOIN laa_ybjdkb   k ON k.jd_id = j.jd_id
        WHERE
            1 = 1
            AND j.jdlb_id IN (
                02
            )
            AND s.zjlx IN (
                01
            )
            AND j.jbjg_id IN (
                14
            )
            AND EXISTS (
                SELECT
                    1
                FROM
                    laa_cjjd_ry   c,
                    laa_apjd_cz   a,
                    laa_jdcc      cc
                WHERE
                    c.jd_id = j.jd_id
                    AND c.jdcz_id = a.jdcz_id
                    AND c.cc_id = cc.cc_id
                    AND TO_CHAR(cc.jdrq, 'yyyy-mm-dd') >= '2025-12-16'
            )
            AND EXISTS (
                SELECT
                    1
                FROM
                    laa_cjjd_ry   c,
                    laa_apjd_cz   a,
                    laa_jdcc      cc
                WHERE
                    c.jd_id = j.jd_id
                    AND c.jdcz_id = a.jdcz_id
                    AND c.cc_id = cc.cc_id
                    AND TO_CHAR(cc.jdrq, 'yyyy-mm-dd') <= '2025-12-16'
            )
        ORDER BY
            j.jdbh
    ) t

时间区间查询条件有问题，且字段加函数用不到相关索引，开发已在修改中


```

## 20251216

### sql1 

```

```

