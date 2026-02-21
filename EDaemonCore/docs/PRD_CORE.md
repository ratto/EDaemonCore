# PRD — EDaemonCore (Engine)
## Biblioteca Core do Sistema Daemon

**Versão:** 1.0  
**Data:** Fevereiro de 2026  
**Status:** Especificação  
**Projeto:** `EDaemonCore`

---

## 1. Visão Geral

### 1.1 Definição

O **EDaemonCore** é uma biblioteca de classes C#.NET (.NET 10) que implementa a *engine* do **Sistema Daemon**. Trata-se de um componente agnóstico de infraestrutura, focado exclusivamente na orquestração de regras de RPG baseadas em testes probabilísticos (d100) e cálculos de margens de sucesso/falha.

### 1.2 Propósito Estratégico

A biblioteca serve como núcleo portável da engine, permitindo:
- **Consumo por múltiplos clientes:** `EDaemonWebServer`, futuros clientes multiplataforma (Meta 2)
- **Isolamento de regras de negócio:** independente de banco de dados, HTTP, ou frameworks de UI
- **Testabilidade máxima:** cobertura ≥ 80% com testes unitários e de integração
- **Extensibilidade controlada:** via padrão Hexagonal, suportando novos Use Cases sem quebra de contrato

---

## 2. Escopo e Responsabilidades

### 2.1 O que está incluído

- **Portas (Ports):** Interfaces públicas que definem contratos de entrada/saída
- **Casos de Uso (UseCases):** Orquestração de fluxos de negócio (ex: testar perícia, calcular sucesso)
- **Serviços (Services):** Lógica de cálculo das regras do Sistema Daemon (rolagens, margens, modificadores)
- **Entidades de Domínio:** Value Objects e DTOs para representar conceitos do jogo (atributos, perícias, resultados)
- **Eventos de Domínio:** Registro de cada etapa de processamento via Event Sourcing

### 2.2 O que **NÃO** está incluído

- Acesso a banco de dados (responsabilidade do consumidor via Ports)
- Endpoints HTTP ou controladores REST (responsabilidade do `EDaemonWebServer`)
- Frameworks de UI ou padrões de apresentação (responsabilidade do Frontend)
- Persistência de estado (apenas processamento imutável de entradas)
- Autenticação, autorização ou gerenciamento de sessão

---

## 3. Arquitetura Técnica

### 3.1 Padrão Arquitetural

**Padrão Principal:** Arquitetura Hexagonal (Ports & Adapters) + Event Sourcing  
**Referências Técnicas:**
- Herberto Graça — [DDD, Hexagonal, Onion, Clean, CQRS](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/)
- Oskar Dudycz — [EventSourcing.NetCore](https://github.com/oskardudycz/EventSourcing.NetCore)
- ByteHide — [Hexagonal Architecture in C#](https://www.bytehide.com/blog/hexagonal-architectural-pattern-in-c-full-guide-2024)

### 3.2 Estrutura de Diretórios

```
EDaemonCore/
├── Ports/                         # Interfaces públicas — Contratos com o exterior
│   ├── Inputs/                    # Portas primárias (input)
│   │   ├── ISkillTestPort.cs      # Contrato para execução de testes de perícia
│   │   └── ...
│   └── Outputs/                   # Portas secundárias (output)
│       ├── IEventLogPort.cs       # Porta de log de eventos (Event Sourcing)
│       └── ...
│
├── UseCases/                      # Orquestração de casos de uso
│   ├── RollSkillTestUseCase.cs    # UC: Executar teste de perícia
│   └── ...
│
├── Services/                      # Lógica de negócio das regras do Sistema Daemon
│   ├── SkillRollService.cs        # Serviço: Rolagem de perícia (d100)
│   ├── SuccessMarginService.cs    # Serviço: Cálculo de margem de sucesso/falha
│   ├── AttributeModifierService.cs# Serviço: Aplicação de modificadores de atributo
│   └── ...
│
├── Domain/                        # Entidades e conceitos de domínio
│   ├── Models/
│   │   ├── SkillModel.cs          # Entidade: Perícia
│   │   ├── RollResultModel.cs     # Entidade: Resultado de rolagem
│   │   ├── CharacterAttributeModel.cs # Entidade: Atributo de personagem
│   │   └── ...
│   │
│   └── Events/
│       ├── SkillRolledEvent.cs    # Evento: "Perícia foi rolada"
│       ├── SuccessMarginCalculatedEvent.cs # Evento: "Margem de sucesso calculada"
│       └── ...
│
└── docs/
    └── (documentação interna da engine)
```

---

## 4. Componentes Principais

### 4.1 Portas (Ports)

**Função:** Definem contratos públicos entre a engine e seus consumidores. Garantem isolamento e portabilidade.

#### 4.1.1 Portas de Entrada (Input Ports)

Interfaces que o consumidor implementa para passar dados à engine:

```csharp
// Exemplo: ISkillTestPort.cs
public interface ISkillTestPort
{
    void ExecuteSkillTest(
        string characterId,
        string skillId,
        Dictionary<string, int> modifiers
    );
}
```

**Responsabilidades:**
- Receber requisições de cálculo
- Validar entrada de dados
- Disparar a execução de Use Cases

#### 4.1.2 Portas de Saída (Output Ports)

Interfaces que a engine publica para o consumidor:

```csharp
// Exemplo: IEventLogPort.cs
public interface IEventLogPort
{
    void LogEvent(DomainEvent @event);
}

// Exemplo: ISkillRepositoryPort.cs
public interface ISkillRepositoryPort
{
    Skill GetSkillById(string skillId);
    IEnumerable<Skill> GetAllSkills();
}
```

**Responsabilidades:**
- Publicar eventos de negócio (Event Sourcing)
- Solicitar dados (repositórios)
- Comunicar erros ou resultados

### 4.2 Casos de Uso (UseCases)

**Função:** Orquestram fluxos completos de negócio, coordenando Services e publicando eventos.

**Estrutura padrão de um UseCase:**
```csharp
public class RollSkillTestUseCase
{
    private readonly ISkillRepositoryPort _skillRepository;
    private readonly IEventLogPort _eventLog;
    private readonly SkillRollService _skillRollService;
    private readonly SuccessMarginService _successMarginService;

    public RollSkillTestUseCase(
        ISkillRepositoryPort skillRepository,
        IEventLogPort eventLog,
        SkillRollService skillRollService,
        SuccessMarginService successMarginService)
    {
        _skillRepository = skillRepository;
        _eventLog = eventLog;
        _skillRollService = skillRollService;
        _successMarginService = successMarginService;
    }

    public SkillTestResult Execute(SkillTestRequest request)
    {
        // 1. Buscar dados via porta
        var skill = _skillRepository.GetSkillById(request.SkillId);

        // 2. Delegar cálculos aos Services
        var rollResult = _skillRollService.Roll(skill, request.Modifiers);
        _eventLog.LogEvent(new SkillRolledEvent(skill.Id, rollResult.Value));

        var margin = _successMarginService.Calculate(rollResult, skill.Difficulty);
        _eventLog.LogEvent(new SuccessMarginCalculatedEvent(margin));

        // 3. Retornar resultado completo
        return new SkillTestResult
        {
            Skill = skill,
            RollValue = rollResult.Value,
            SuccessMargin = margin,
            IsSuccess = margin >= 0,
            Events = _eventLog.GetEvents() // Event Sourcing
        };
    }
}
```

**Regra essencial:** Cada interação com um Service deve gerar um evento para a porta de log.

### 4.3 Serviços (Services)

**Função:** Implementam a lógica pura das regras do Sistema Daemon. Sem dependências de infraestrutura.

#### 4.3.1 SkillRollService

Responsável por rolagens de perícia (d100):

```csharp
public class SkillRollService
{
    private readonly Random _random;

    public RollResult Roll(Skill skill, Dictionary<string, int> modifiers)
    {
        int roll = _random.Next(1, 101); // d100
        int finalValue = roll + modifiers.Values.Sum();
        
        return new RollResult
        {
            BaseRoll = roll,
            Modifiers = modifiers,
            Value = finalValue
        };
    }
}
```

#### 4.3.2 SuccessMarginService

Calcula margem de sucesso/falha baseada na dificuldade da perícia:

```csharp
public class SuccessMarginService
{
    public int Calculate(RollResult rollResult, int skillDifficulty)
    {
        return rollResult.Value - skillDifficulty;
    }
}
```

#### 4.3.3 AttributeModifierService

Aplica modificadores baseados em atributos do personagem:

```csharp
public class AttributeModifierService
{
    public Dictionary<string, int> CalculateModifiers(
        CharacterAttributes attributes,
        Skill skill)
    {
        // Lógica de cálculo baseada no Sistema Daemon
        // Ex: Força afeta testes de Atletismo
    }
}
```

### 4.4 Domínio (Domain)

#### 4.4.1 Entidades de Domínio

```csharp
// Skill.cs
public class Skill
{
    public string Id { get; set; }
    public string Name { get; set; }
    public int BaseDifficulty { get; set; }
    public string AssociatedAttribute { get; set; }
    public string Description { get; set; }
}

// RollResult.cs
public class RollResult
{
    public int BaseRoll { get; set; }
    public Dictionary<string, int> Modifiers { get; set; }
    public int Value { get; set; }
}

// SkillTestResult.cs
public class SkillTestResult
{
    public Skill Skill { get; set; }
    public int RollValue { get; set; }
    public int SuccessMargin { get; set; }
    public bool IsSuccess { get; set; }
    public List<DomainEvent> Events { get; set; }
}
```

#### 4.4.2 Eventos de Domínio (Domain Events)

```csharp
// DomainEvent.cs (Classe base)
public abstract class DomainEvent
{
    public DateTime OccurredAt { get; set; } = DateTime.UtcNow;
    public string UseCaseId { get; set; }
}

// SkillRolledEvent.cs
public class SkillRolledEvent : DomainEvent
{
    public string SkillId { get; set; }
    public int RollValue { get; set; }
}

// SuccessMarginCalculatedEvent.cs
public class SuccessMarginCalculatedEvent : DomainEvent
{
    public int Margin { get; set; }
    public bool IsSuccess { get; set; }
}
```

---

## 5. Regras de Design e Restrições

### 5.1 Restrições Arquiteturais (RNF-01)

**Princípio:** O Core **nunca importa** nada de fora de seus próprios namespaces.

✅ **Permitido:**
- Imports de `EDaemonCore.*`
- Tipos nativos do C# (`System.*`)
- Interfaces públicas (Ports)

❌ **Proibido:**
- `using Microsoft.AspNetCore.*` (HTTP)
- `using Microsoft.EntityFrameworkCore.*` (BD)
- `using Quasar.*` ou qualquer framework de UI
- Dependências NuGet de infraestrutura

### 5.2 Fluxo Obrigatório de Portas

Cada UseCase segue o padrão:

```
[Porta de Entrada] → UseCase → Service → [Porta de Saída]
```

**Exemplos:**
1. UseCase recebe `ISkillTestPort` (entrada)
2. UseCase chama `SkillRollService` (cálculo)
3. UseCase publica `SkillRolledEvent` via `IEventLogPort` (saída)

### 5.3 Imutabilidade de Resultados

Todos os resultados retornados pelos UseCases devem ser imutáveis ou cópias profundas:

```csharp
// ❌ Errado: modifica estado compartilhado
public RollResult Execute(Request request)
{
    var result = new RollResult();
    result.Value = 50; // mutação
    return result;
}

// ✅ Correto: retorna nova instância
public RollResult Execute(Request request)
{
    return new RollResult { Value = 50 };
}
```

---

## 6. Fluxo de Dados — Exemplo Completo

### Caso de Uso: Executar Teste de Perícia

```
1. [Consumidor — EDaemonWebServer]
   Service chama: new RollSkillTestUseCase(...).Execute(request)

2. [UseCase — Orquestração]
   RollSkillTestUseCase.Execute()
   ├─ Busca Skill via _skillRepository.GetSkillById()
   │   └─ Publica evento: SkillLoadedEvent
   │
   ├─ Delega rolagem para _skillRollService.Roll()
   │   └─ Retorna RollResult { BaseRoll=45, Modifiers={...}, Value=50 }
   │   └─ Publica evento: SkillRolledEvent
   │
   ├─ Calcula margem via _successMarginService.Calculate()
   │   └─ Retorna margin = 50 - 40 = +10 (sucesso)
   │   └─ Publica evento: SuccessMarginCalculatedEvent
   │
   └─ Retorna SkillTestResult com todos os eventos

3. [Consumidor — EDaemonWebServer]
   Service recebe SkillTestResult
   ├─ Converte para DTO
   ├─ Registra eventos em log (Serilog)
   └─ Retorna ao Controller (HTTP 200)

4. [Frontend]
   Recebe JSON com resultado + etapas
   └─ Exibe passo a passo do teste
```

---

## 7. Strategy de Testes

### 7.1 Estrutura de Testes

```
EDaemonCore.Tests/
├── UseCases/
│   └── RollSkillTestUseCase.Tests.cs
│
├── Services/
│   ├── SkillRollService.Tests.cs
│   ├── SuccessMarginService.Tests.cs
│   └── AttributeModifierService.Tests.cs
│
└── Integration/
    └── RollSkillTestUseCaseIntegration.Tests.cs
```

### 7.2 Testes Unitários

**Ferramenta:** xUnit  
**Padrão:** Arrange-Act-Assert (AAA)

```csharp
[Fact]
public void SkillRollService_Roll_ShouldReturnValueBetween1And100()
{
    // Arrange
    var service = new SkillRollService();
    var skill = new Skill { Id = "sk_001", Name = "Ataque" };
    var modifiers = new Dictionary<string, int> { { "Força", 5 } };

    // Act
    var result = service.Roll(skill, modifiers);

    // Assert
    Assert.InRange(result.Value, 1, 105); // 1-100 + modificador
}
```

### 7.3 Testes de Integração

Validam fluxos completos entre múltiplos Services e Portas:

```csharp
[Fact]
public void RollSkillTestUseCase_Execute_ShouldPublishAllEvents()
{
    // Arrange
    var mockSkillRepository = new Mock<ISkillRepositoryPort>();
    var mockEventLog = new Mock<IEventLogPort>();
    var rollService = new SkillRollService();
    var marginService = new SuccessMarginService();

    var useCase = new RollSkillTestUseCase(
        mockSkillRepository.Object,
        mockEventLog.Object,
        rollService,
        marginService);

    // Act
    var result = useCase.Execute(new SkillTestRequest { /* ... */ });

    // Assert
    mockEventLog.Verify(p => p.LogEvent(It.IsAny<SkillRolledEvent>()), Times.Once);
    mockEventLog.Verify(p => p.LogEvent(It.IsAny<SuccessMarginCalculatedEvent>()), Times.Once);
    Assert.True(result.IsSuccess);
}
```

### 7.4 Cobertura de Testes

**Alvo:** ≥ 80% em UseCases e Services  
**Métricas:** Utilizar Coverlet para relatórios automatizados

```bash
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover
```

---

## 8. Dependências e Stack Técnico

### 8.1 Dependências Permitidas

| Dependência | Versão | Uso | Justificativa |
|---|---|---|---|
| **.NET** | 10.0 | Target Framework | Latest LTS com features modernas |

**Nota:** O projeto deve ter **ZERO** dependências NuGet externas na fase inicial (MVP).

### 8.2 Stack Técnico

```
Linguagem:     C# 13+ (latest)
Framework:     .NET 10.0
Padrão:        Hexagonal + Event Sourcing
Testes:        xUnit
Cobertura:     Coverlet
```

---

## 9. Contrato Público (API Pública)

### 9.1 Namespaces Públicos

Apenas estes namespaces podem ser importados por consumidores:

```csharp
using EDaemonCore.Ports.Inputs;     // Portas de entrada
using EDaemonCore.Ports.Outputs;    // Portas de saída
using EDaemonCore.UseCases;         // Casos de uso
using EDaemonCore.Domain.Models;    // DTOs de domínio
```

### 9.2 Interfaces Públicas Esperadas

O consumidor (`EDaemonWebServer`) espera implementar:

```csharp
// Interfaces que o consumidor implementa (adapters)
public interface ISkillRepositoryPort { /* ... */ }
public interface IEventLogPort { /* ... */ }

// Classes que o consumidor instancia
public class RollSkillTestUseCase { /* ... */ }
public class SkillRollService { /* ... */ }
```

---

## 10. Critérios de Aceitação (CA) para EDaemonCore

| ID | Critério | Como Validar |
|---|---|---|
| CA-CR-01 | Core compila sem nenhuma dependência de infraestrutura | Auditoria de `.csproj` |
| CA-CR-02 | Cada UseCase publica ≥ 2 eventos via IEventLogPort | Testes de integração |
| CA-CR-03 | Cobertura de testes ≥ 80% em Services + UseCases | Relatório Coverlet |
| CA-CR-04 | SkillRollService implementa d100 (1-100) com modificadores | Testes unitários |
| CA-CR-05 | SuccessMarginService calcula corretamente margem de sucesso/falha | Testes unitários |
| CA-CR-06 | Nenhuma exceção não-tratada escapa do UseCase | Testes de exceção |
| CA-CR-07 | Event Sourcing funciona: sequência de eventos é rastreável | Testes de integração |
| CA-CR-08 | Todas as Portas podem ser mockadas para testes | Verificação xUnit com Moq |

---

## 11. Próximas Fases (Roadmap)

### Meta 1 (Atual)
- ✅ Arquitetura Hexagonal
- ✅ Caso de Uso: RollSkillTest
- ✅ Serviços: Roll, SuccessMargin, AttributeModifier
- ✅ Event Sourcing básico
- ✅ Testes unitários e de integração

### Meta 2 (Multiplataforma)
- Exportação para JavaScript/TypeScript (Wasm ou Node.js)
- Suporte a cenários mais complexos

### Meta 3 (Jogo Completo)
- Game loop completo
- Persistência de personagens
- Multiplayer (TBD)

---

## 12. Referências e Recursos

| Tema | Referência |
|---|---|
| Hexagonal Architecture | https://www.bytehide.com/blog/hexagonal-architectural-pattern-in-c-full-guide-2024 |
| Event Sourcing | https://github.com/oskardudycz/EventSourcing.NetCore |
| DDD em C# | https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/ |
| xUnit Testing | https://xunit.net/docs/getting-started/netcore |
| Moq Mocking | https://github.com/moq/moq4 |
| .NET 10 | https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-10 |

---

*Este documento é a especificação técnica do projeto EDaemonCore e deve ser mantido atualizado conforme a implementação evolui.*
