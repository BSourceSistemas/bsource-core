# Guia de validação, `Result` e respostas HTTP

## Objetivo

Este guia define o contrato entre API e Application para operações de negócio. Ele pode ser levado a qualquer repositório .NET que use MediatR e FluentValidation: os nomes podem mudar, mas as responsabilidades e o fluxo devem permanecer.

```text
HTTP Request → API Request → Command/Query → Validator (pipeline) → Handler
                                                          ↓
                                               Result<TDto> ou Result
                                                          ↓
                                      Controller → resposta HTTP padronizada
```

## 1. Contrato `Result`

Os handlers retornam falhas esperadas como valores, não como exceções. `Result` representa uma operação sem conteúdo; `Result<T>` representa uma operação cujo sucesso contém um valor `T`.

```csharp
public sealed record Error(
    string Code,
    string Message,
    ErrorType Type,
    IReadOnlyDictionary<string, string[]>? Details = null);

public class Result
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public IReadOnlyList<Error> Errors { get; }

    public static Result Success();
    public static Result Fail(Error error);
    public static Result Fail(IEnumerable<Error> errors);
}

public sealed class Result<T> : Result
{
    public T? Value { get; }

    public static Result<T> Success(T value);
    public static Result<T> Fail(Error error);
    public static Result<T> Fail(IEnumerable<Error> errors);
}
```

### Regras de uso

- Use `Result.Success()` em comandos sem payload, normalmente `Delete` ou alteração de associação.
- Use `Result<T>.Success(valor)` quando o handler retornar um DTO ou coleção.
- Use `Result<T>.Fail(new Error(...))` ou `Result.Fail(new Error(...))` para falhas de negócio previsíveis.
- Nunca acesse `result.Value` sem antes garantir sucesso. Em um caminho de sucesso já protegido por `if (!result.IsSuccess)`, `result.Value!` expressa essa garantia ao compilador.
- Propague vários erros com `Fail(errors)` quando o caso de uso já os possuir.
- Exceções ficam restritas a falhas inesperadas, técnicas, ou a respostas que não utilizam `Result`; o handler global as normaliza.

### Criação de erros

```csharp
return Result<EntityDto>.Fail(new Error(
    "Entity.NameConflict",                // identificador estável e em inglês
    $"Já existe um registro com o nome '{request.Name}'", // mensagem ao consumidor, pt-BR
    ErrorType.Conflict));
```

Escolha `ErrorType` pela semântica: `Validation` (400), `BadRequest` (400), `NotFound` (404), `Conflict` (409), `Forbidden` (403), `Unauthorized` (401), `BusinessRule` (422) e `Unexpected` (500).

## 2. Validators na Application

Cada Command ou Query com validação declarativa possui um validator FluentValidation na mesma feature. O validator herda `AbstractValidator<TRequest>` e contém apenas regras de formato e pré-condições que não dependem de consultar estado de negócio.

```csharp
public sealed class CreateEntityCommandValidator
    : AbstractValidator<CreateEntityCommand>
{
    public CreateEntityCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MaximumLength(200).WithMessage("Name must not exceed 200 characters");

        RuleFor(x => x.Description)
            .MaximumLength(500).WithMessage("Description must not exceed 500 characters")
            .When(x => x.Description is not null);
    }
}
```

O registro `AddValidatorsFromAssembly(...)` descobre validators, e o `ValidationBehavior<TRequest, TResponse>` do MediatR executa todos os validators antes do handler. Quando o retorno é `Result` ou `Result<T>`, o pipeline agrupa as falhas por propriedade no `Error.Details` e devolve `Result.Fail` com `ErrorType.Validation`; portanto, controllers não chamam `Validate` manualmente.

Validações que exigem repositório, contexto do usuário, unicidade, estado atual ou regra de domínio ficam no handler (ou no Domain, quando forem invariantes). Elas também retornam `Result.Fail(new Error(...))`.

## 3. Handler: falha previsível e sucesso

```csharp
public async Task<Result<EntityDto>> Handle(
    CreateEntityCommand request,
    CancellationToken cancellationToken)
{
    if (!_currentUserContext.OrganizationId.HasValue)
    {
        return Result<EntityDto>.Fail(new Error(
            "Entity.OrganizationNotFound",
            "A organização do usuário autenticado não foi encontrada",
            ErrorType.Validation));
    }

    var existing = await _entityRepository.GetByNameAsync(
        _currentUserContext.OrganizationId.Value, request.Name, cancellationToken);
    if (existing is not null)
    {
        return Result<EntityDto>.Fail(new Error(
            "Entity.NameConflict",
            $"Já existe um registro com o nome '{request.Name}'",
            ErrorType.Conflict));
    }

    // Persiste e obtém o DTO completo que será devolvido.
    var result = await _mediator.Send(new GetEntityByIdQuery(entity.Id), cancellationToken);
    return Result<EntityDto>.Success(result.Value!);
}
```

Para `Create` e `Update`, é recomendável reconsultar a entidade após persistir, para produzir o DTO completo. A expressão `Result<T>.Success(result.Value!)` só é válida quando a consulta anterior tem sucesso garantido pelo fluxo. Se isso não puder ser garantido, a falha deve ser propagada:

```csharp
if (!result.IsSuccess)
    return Result<EntityDto>.Fail(result.Errors);

return Result<EntityDto>.Success(result.Value!);
```

## 4. Controller: adaptador HTTP fino

O controller recebe o contrato HTTP, cria Command/Query, chama o mediator, converte falhas e mapeia DTOs para contratos de resposta. Ele não valida manualmente, não acessa repositórios e não expõe DTOs da Application.

```csharp
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateEntityRequest request)
{
    var result = await _mediator.Send(new CreateEntityCommand(
        request.Name,
        request.Description));

    if (!result.IsSuccess)
        return result.ToProblemDetails(this);

    return Ok(CollectionResponse<EntityResponse>.From(
        new EntityResponse(result.Value!)));
}
```

O formato de sucesso é uniforme: inclusive um item único é encapsulado por `CollectionResponse<TResponse>`, com `Results` e `Total`. O construtor de `EntityResponse` é responsável por mapear o DTO da Application.

| Operação | Sucesso no controller |
|---|---|
| Create, Update, Get | `Ok(CollectionResponse<TResponse>.From(new TResponse(result.Value!)))` |
| List | `Ok(CollectionResponse<TResponse>.From(result.Value!.Results.Select(x => new TResponse(x))))` |
| Delete e comandos sem conteúdo | `NoContent()` |

Em caso de falha, use sempre a extensão centralizada:

```csharp
if (!result.IsSuccess)
    return result.ToProblemDetails(this);
```

`ToProblemDetails` constrói `ErrorResponse` e traduz o primeiro `ErrorType` para o status HTTP. Para erros de validação com `Details`, `ErrorResponse` transforma cada propriedade e mensagem em um item de `Errors`, permitindo que o cliente associe a falha ao campo correto.

## 5. Checklist de adoção em outro repositório

1. Coloque `Result`, `Result<T>`, `Error` e `ErrorType` em uma biblioteca compartilhada acessível à Application e à API.
2. Faça Commands e Queries implementarem `IRequest<Result>` ou `IRequest<Result<TDto>>`.
3. Registre FluentValidation por assembly e o pipeline MediatR que converte falhas em `Result.Fail`.
4. Centralize a tradução `ErrorType → HTTP` em uma extensão de controller e defina um único contrato de erro.
5. Mantenha cada controller como adaptador: request → command/query → `IsSuccess` → response.
6. Documente explicitamente qualquer endpoint que precise fugir do envelope de sucesso ou do status de sucesso padrão.
