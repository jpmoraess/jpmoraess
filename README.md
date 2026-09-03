### Hello, I'm João Pedro ✌

Experienced Backend Developer passionate about building clean, efficient, and scalable solutions. I specialize in Java and the Spring ecosystem, and have recently expanded my expertise with Kotlin and Golang to stay current with modern development trends.

I thrive on solving complex problems, designing intuitive and robust APIs, and creating systems that are both reliable and maintainable. I value code quality, performance, and architecture, and I'm constantly exploring new technologies and approaches to improve my work.

Driven by challenges, I enjoy working on high-impact systems, contributing to business goals through well-structured, resilient backend Segunda versão 
WITH vinculos_ativos AS (
    SELECT
        v.id_conta,
        v.id_pessoa
    FROM datamesh.vinculos v
    GROUP BY
        v.id_conta,
        v.id_pessoa
    HAVING
        MAX(
            CASE
                WHEN v.status = 'ATIVO' THEN 1
                ELSE 0
            END
        ) = 1
        AND
        MAX(
            CASE
                WHEN v.status = 'INATIVO' THEN 1
                ELSE 0
            END
        ) = 0
),

encerramentos AS (
    SELECT DISTINCT
        e.id_conta,
        e.id_pessoa
    FROM datamesh.encerramento_conta e
    WHERE
           (
               e.ano = '2025'
               AND e.mes IN (
                   '08',
                   '09',
                   '10',
                   '11',
                   '12'
               )
           )
        OR (
               e.ano = '2026'
               AND e.mes IN (
                   '01',
                   '02',
                   '03',
                   '04',
                   '05',
                   '06',
                   '07',
                   '08'
               )
           )
)

SELECT
    v.id_conta,
    v.id_pessoa
FROM vinculos_ativos v
INNER JOIN encerramentos e
    ON e.id_conta = v.id_conta
   AND e.id_pessoa = v.id_pessoa;

