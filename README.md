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
   AND e.id_Boa, esse cenário é um candidato clássico para combinar Strategy (cada regra é intercambiável) + Composite/Orchestrator (o motor agrega o resultado) + Strategy também para a forma de execução (sequencial vs paralelo fail-fast vs paralelo acumulando). Não precisa de Chain of Responsibility puro porque você quer tanto short-circuit quanto acumulação, dependendo de config — então isolar a "estratégia de execução" separada da "regra em si" é o que te dá essa flexibilidade.
E o pulo do gato aqui, já que você está no JDK 21 e não pode usar WebFlux/suspend: use Virtual Threads. Elas te dão concorrência "de graça" com código 100% bloqueante (Feign incluso), sem precisar reescrever nada como reativo. Isso resolve exatamente o requisito de "assíncrono sem WebFlux".
Estrutura geral
EligibilityRule (interface)          -> Strategy: cada regra de negócio
RuleResult (sealed class)            -> Approved / Rejected
RuleExecutor (interface)             -> Strategy: como as regras são executadas
  ├── SequentialRuleExecutor
  ├── ParallelFailFastRuleExecutor
  └── ParallelAccumulateRuleExecutor
EligibilityEngine                    -> Orquestrador: ordena/filtra regras e delega ao executor certo
EligibilityProperties                -> @ConfigurationProperties com modo e ordem
1. Contrato da regra
interface EligibilityRule {
    val code: String
    fun evaluate(context: EligibilityContext): RuleResult
}

data class EligibilityContext(
    val customerId: String,
    val attributes: Map<String, Any> = emptyMap()
)

sealed class RuleResult {
    data class Approved(val ruleCode: String) : RuleResult()
    data class Rejected(val ruleCode: String, val reason: String) : RuleResult()
}
2. Regra concreta (exemplo com Feign bloqueante)
@Component
class CreditScoreRule(
    private val creditClient: CreditFeignClient
) : EligibilityRule {

    override val code = "creditScoreRule"

    override fun evaluate(context: EligibilityContext): RuleResult {
        val response = creditClient.getScore(context.customerId) // chamada bloqueante normal
        return if (response.score >= 600) {
            RuleResult.Approved(code)
        } else {
            RuleResult.Rejected(code, "Score de crédito insuficiente: ${response.score}")
        }
    }
}
Cada nova regra = uma nova classe @Component implementando EligibilityRule. Zero mudança no motor.
3. Properties (ordem + modo de execução)
@ConfigurationProperties(prefix = "eligibility")
data class EligibilityProperties(
    var executionMode: ExecutionMode = ExecutionMode.SEQUENTIAL,
    var rules: Map<String, RuleConfig> = emptyMap()
) {
    data class RuleConfig(
        var enabled: Boolean = true,
        var order: Int = Int.MAX_VALUE
    )
}

enum class ExecutionMode { SEQUENTIAL, PARALLEL_FAIL_FAST, PARALLEL_ACCUMULATE }
eligibility:
  execution-mode: PARALLEL_FAIL_FAST
  rules:
    creditScoreRule:
      enabled: true
      order: 1
    fraudCheckRule:
      enabled: true
      order: 2
    blacklistRule:
      enabled: false
      order: 3
4. Executores (as duas abordagens que você pensou, como estratégias plugáveis)
interface RuleExecutor {
    fun execute(rules: List<EligibilityRule>, context: EligibilityContext): EligibilityResponse
}

data class EligibilityResponse(
    val eligible: Boolean,
    val reasons: List<String> = emptyList()
)
Sequencial, acumulando motivos:
class SequentialRuleExecutor : RuleExecutor {
    override fun execute(rules: List<EligibilityRule>, context: EligibilityContext): EligibilityResponse {
        val reasons = rules
            .map { it.evaluate(context) }
            .filterIsInstance<RuleResult.Rejected>()
            .map { it.reason }
        return EligibilityResponse(eligible = reasons.isEmpty(), reasons = reasons)
    }
}
Paralelo, fail-fast (primeira que falhar cancela o resto):
class ParallelFailFastRuleExecutor(
    private val executor: ExecutorService
) : RuleExecutor {

    override fun execute(rules: List<EligibilityRule>, context: EligibilityContext): EligibilityResponse {
        if (rules.isEmpty()) return EligibilityResponse(eligible = true)

        val completionService = ExecutorCompletionService<RuleResult>(executor)
        val futures = rules.map { rule -> completionService.submit(Callable { rule.evaluate(context) }) }

        try {
            repeat(rules.size) {
                val result = completionService.take().get()
                if (result is RuleResult.Rejected) {
                    futures.forEach { it.cancel(true) }
                    return EligibilityResponse(eligible = false, reasons = listOf(result.reason))
                }
            }
        } catch (ex: ExecutionException) {
            futures.forEach { it.cancel(true) }
            throw ex.cause ?: ex
        }

        return EligibilityResponse(eligible = true)
    }
}
Paralelo, acumulando todos os motivos (roda tudo, sem cancelar):
class ParallelAccumulateRuleExecutor(
    private val executor: ExecutorService
) : RuleExecutor {
    override fun execute(rules: List<EligibilityRule>, context: EligibilityContext): EligibilityResponse {
        val futures = rules.map { rule -> CompletableFuture.supplyAsync({ rule.evaluate(context) }, executor) }
        val reasons = futures.map { it.join() }
            .filterIsInstance<RuleResult.Rejected>()
            .map { it.reason }
        return EligibilityResponse(eligible = reasons.isEmpty(), reasons = reasons)
    }
}
5. O motor (orquestrador)
interface EligibilityEngine {
    fun evaluate(context: EligibilityContext): EligibilityResponse
}

@Service
class DefaultEligibilityEngine(
    rules: List<EligibilityRule>,
    private val properties: EligibilityProperties,
    private val executors: Map<ExecutionMode, RuleExecutor>
) : EligibilityEngine {

    private val orderedRules: List<EligibilityRule> = rules
        .filter { properties.rules[it.code]?.enabled ?: true }
        .sortedBy { properties.rules[it.code]?.order ?: Int.MAX_VALUE }

    override fun evaluate(context: EligibilityContext): EligibilityResponse {
        val executor = executors[properties.executionMode]
            ?: error("Modo de execução não suportado: ${properties.executionMode}")
        return executor.execute(orderedRules, context)
    }
}
6. Configuração (Virtual Threads + wiring dos executores)
@Configuration
class ExecutorConfig {
    @Bean
    fun eligibilityExecutor(): ExecutorService = Executors.newVirtualThreadPerTaskExecutor()
}

@Configuration
class RuleExecutorConfig {

    @Bean
    fun sequentialExecutor() = SequentialRuleExecutor()

    @Bean
    fun parallelFailFastExecutor(eligibilityExecutor: ExecutorService) =
        ParallelFailFastRuleExecutor(eligibilityExecutor)

    @Bean
    fun parallelAccumulateExecutor(eligibilityExecutor: ExecutorService) =
        ParallelAccumulateRuleExecutor(eligibilityExecutor)

    @Bean
    fun ruleExecutorsMap(
        sequentialExecutor: SequentialRuleExecutor,
        parallelFailFastExecutor: ParallelFailFastRuleExecutor,
        parallelAccumulateExecutor: ParallelAccumulateRuleExecutor
    ): Map<ExecutionMode, RuleExecutor> = mapOf(
        ExecutionMode.SEQUENTIAL to sequentialExecutor,
        ExecutionMode.PARALLEL_FAIL_FAST to parallelFailFastExecutor,
        ExecutionMode.PARALLEL_ACCUMULATE to parallelAccumulateExecutor
    )
}
Vale também habilitar spring.threads.virtual.enabled=true no application.properties (Spring Boot 3.2+), assim o próprio Tomcat processa cada request numa virtual thread — combinando bem com o executor acima.
7. Controller (sem suspend, tudo bloqueante normal)
@RestController
@RequestMapping("/eligibility")
class EligibilityController(private val engine: EligibilityEngine) {

    @PostMapping("/evaluate")
    fun evaluate(@RequestBody request: EligibilityRequest): ResponseEntity<EligibilityResponse> {
        val context = EligibilityContext(request.customerId, request.attributes)
        return ResponseEntity.ok(engine.evaluate(context))
    }
}
Um ponto de atenção honesto sobre o "cancelamento"
future.cancel(true) interrompe a thread, mas se o cliente HTTP por trás do Feign (ex.: URLConnection padrão) não trata InterruptedException durante I/O bloqueante, a chamada de rede pode continuar em andamento no backend mesmo cancelada do seu lado — você só descarta o resultado mais cedo. Se quiser cancelamento "de verdade" (abortar o socket), vale configurar o Feign com client OkHttp ou Apache HttpClient5, que respeitam interrupção/cancelamento de forma mais confiável. Para a maioria dos casos de fail-fast, "parar de esperar e responder rápido" já resolve o requisito de negócio, mesmo que a chamada remota termine sozinha em background.
Esse desenho te dá exatamente as duas abordagens que você mencionou, configuráveis via properties, sem tocar no motor quando uma regra nova entrar — só criar a classe e registrar no YAML.


