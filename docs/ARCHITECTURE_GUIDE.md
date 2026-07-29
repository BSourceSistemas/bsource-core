# ARCHITECTURE_GUIDE

## 1. Objetivo

Este documento define as **diretrizes arquiteturais obrigatórias** para todos os projetos da plataforma.

Ele estabelece **regras normativas**, não sugestões, sobre:

* Estrutura do projeto
* Separação de responsabilidades
* Contratos entre camadas
* Padrões técnicos obrigatórios

Estas diretrizes **independem do domínio de negócio** e devem ser seguidas em todos os sistemas.

---

## 2. Estrutura Padrão do Projeto

Todo projeto **DEVE** seguir a estrutura abaixo:

```
/docs
/docker
/src
 ├── NomeProjeto.API
 ├── NomeProjeto.Domain
 ├── NomeProjeto.Application
 ├── NomeProjeto.Infrastructure
 └── NomeProjeto.Shared
```

---

## 3. Princípios Arquiteturais

3.1 Separação de responsabilidades clara

3.2 Dependências sempre apontam para dentro:

```
API → Application → Domain
API → Application → Infrastructure (via abstrações)
```

3.3 Nenhuma camada externa conhece detalhes internos de outra

3.4 Regras de negócio **NUNCA** dependem de infraestrutura

---

## 4. Padrão de Mediação

4.1 Toda comunicação entre API e Application **DEVE** ocorrer via padrão de mediação (MediatR)

4.2 Controllers **NÃO DEVEM** chamar serviços diretamente

4.3 Application expõe apenas **Commands e Queries**

---

## 5. Padrão Result-Driven

5.1 Toda comunicação entre API e Application **DEVE** trafegar via `Result` ou `Result<T>` (definidos em Shared)
5.2 Handlers **NUNCA** lançam exceções para erros de negócio — usam `Result.Fail`
5.3 Tipos:
  - `Result` — operações sem retorno de dados (Delete)
  - `Result<T>` — operações que retornam um item (Create, Update, Get)
  - `Result<CollectionResult<T>>` — operações que retornam coleções (List)

> Para o contrato completo, fluxo de FluentValidation, exemplos de `Result<T>.Fail`, `Result<T>.Success(result.Value!)` e adaptação no controller, consulte [RESULT_VALIDATION_CONTROLLER_GUIDE.md](RESULT_VALIDATION_CONTROLLER_GUIDE.md).

---

## 6. Tipos de Retorno por Operação

### 6.1 Retornos do Handler (Application)

| Operação | Handler retorna |
|----------|-----------------|
| Create   | `Result<TDto>`  |
| Update   | `Result<TDto>`  |
| Delete   | `Result`        |
| Get      | `Result<TDto>`  |
| List     | `Result<CollectionResult<TDto>>` |

6.1.1 Para Create/Update, o handler **DEVE** re-consultar via `GetEntityByIdQuery` após persistir, garantindo retorno completo e consistente.  
6.1.2 Para Delete, o handler realiza soft delete (`BaseStatus.Deleted`) e retorna apenas `Result.Success()`.

### 6.2 Retornos do Controller (API)

6.2.1 Todas as respostas de sucesso (item único ou coleção) **DEVEM** ser encapsuladas em `CollectionResponse<TResponse>`  
6.2.2 A API **DEVE** usar os contratos `CollectionResponse<T>` e `ErrorResponse` na camada Contracts/Responses  
6.2.3 O mapeamento DTO → Response **DEVE** ser feito pelo construtor da classe Response  

### 6.3 Padrões de Retorno no Controller

| Operação | Controller retorna |
|----------|-------------------|
| Create   | `CollectionResponse<TResponse>.From(new TResponse(result.Value!))` |
| Update   | `CollectionResponse<TResponse>.From(new TResponse(result.Value!))` |
| Get      | `CollectionResponse<TResponse>.From(new TResponse(result.Value!))` |
| List     | `CollectionResponse<TResponse>.From(result.Value!.Results.Select(x => new TResponse(x)))` |
| Delete   | `NoContent()` |

---

## 7. Nomenclatura de Ações

7.1 Commands:
  - `CreateEntityCommand` — criar 1 ou mais recursos
  - `UpdateEntityCommand` — atualizar 1 ou mais recursos
  - `DeleteEntityCommand` — excluir 1 ou mais recursos
7.2 Queries:
  - `GetEntityByIdQuery` — obter 1 recurso
  - `ListEntitiesQuery` — obter 2 ou mais recursos

---

## 8. Tratamento de Erros

8.1 Erros **DEVEM** ser representados por `Error(Code, Message, ErrorType, Details?)`
8.2 O `ErrorType` **DEVE** ser escolhido pela semântica do erro: `Validation`, `NotFound`, `Conflict`, `Forbidden`, `Unauthorized`, `BusinessRule`, `BadRequest`, `Unexpected`
8.3 A API converte `ErrorType` para HTTP status via `ResultExtensions.ToProblemDetails`:
  - `Validation` / `BadRequest` → 400
  - `NotFound` → 404
  - `Conflict` → 409
  - `Forbidden` → 403
  - `Unauthorized` → 401
  - `BusinessRule` → 422
  - `Unexpected` → 500
8.4 O pipeline MediatR executa validações FluentValidation e converte falhas em `Result.Fail` automaticamente.
8.5 O campo `Error.Code` **DEVE** permanecer em inglês (identificador programático, ex: `"User.NotFound"`)
8.6 O campo `Error.Message` **DEVE** estar sempre em português (pt-BR), pois é o texto apresentado ao consumidor da API

---

## 9. Responsabilidades das Camadas

### 9.1 API

* Atua exclusivamente como **adaptador HTTP**
* Expõe endpoints
* Executa autenticação e autorização
* Mapeia Request → Command / Query
* Mapeia Result → Response

### 9.2 Application

* Implementa casos de uso
* Orquestra fluxos
* Define Commands, Queries e Handlers
* Contém DTOs internos
* Contém Validators (FluentValidation)

### 9.3 Domain

* Contém regras invariantes
* Modela o negócio
* Entidades e Value Objects

### 9.4 Infrastructure

* Persistência
* Integrações externas
* Serviços técnicos

### 9.5 Shared

* Abstrações
* Contratos
* Utilitários transversais

---

## 10. Contratos da Camada API

### 10.1 Princípio

A camada API **NÃO expõe estruturas internas da aplicação**.

Ela atua apenas como adaptador entre HTTP e Application.

---

### 10.2 Regras Obrigatórias

10.2.1 A API **NÃO DEVE** utilizar DTOs da camada Application

10.2.2 A API **DEVE** trabalhar apenas com:

* Request (entrada HTTP)
* Response / Result (saída HTTP)

10.2.3 DTOs pertencem **exclusivamente à camada Application**

10.2.4 Controllers **NÃO DEVEM** acessar Domain ou Infrastructure diretamente

10.2.5 Controllers **DEVEM** apenas:

* Receber Requests
* Autorizar acesso
* Mapear para Commands / Queries
* Delegar para Application
* Mapear resultado para Responses

10.2.6 Todo objeto Response **DEVE** ser uma `class` (não `record`)

10.2.7 Toda classe Response **DEVE** conter ao menos um construtor que receba o DTO correspondente da Application e realize o parse/mapeamento dos campos

10.2.8 O mapeamento DTO → Response fica encapsulado **dentro da própria classe Response**, mantendo o controller limpo

---

### 10.3 Fluxo Arquitetural Oficial

```
HTTP Request
   ↓
API Request
   ↓
Command / Query
   ↓
Application Handler
   ↓
Result<TDto> ou Result<CollectionResult<TDto>>
   ↓
Controller verifica IsSuccess
   ↓
new TResponse(dto)
   ↓
CollectionResponse<TResponse>
   ↓
HTTP Response
```

Fluxo de erro:
```
Handler retorna Result.Fail(Error)
   ↓
Controller detecta IsFailure
   ↓
ToProblemDetails
   ↓
HTTP Error (4xx/5xx)
```

---

## 11. Validações

11.1 Validações de entrada **DEVEM** ser feitas na Application

11.2 O padrão obrigatório é **FluentValidation**

11.3 Validators validam:

* Formato
* Consistência de dados
* Regras de caso de uso

11.4 O Domain contém apenas **invariantes essenciais**

---

## 12. Autenticação e Autorização

12.1 Autenticação baseada em JWT

12.2 Login é o único endpoint público

12.3 Todos os demais endpoints **DEVEM** exigir token válido

12.4 Autorização baseada em **permissões**, nunca roles fixas

12.5 Endpoints protegidos **DEVEM** declarar policies explicitamente

---

## 13. DbContext

13.1 Deve existir:

* Um DbContext de escrita
* Um DbContext somente leitura

13.2 Queries **DEVEM** utilizar o DbContext de leitura

---

## 14. Platform Core

14.1 Funcionalidades transversais obrigatórias:

* Autenticação
* Usuários
* Grupos
* Permissões

14.2 O Platform Core é definido em documentos próprios:

* PLATFORM_CORE_GUIDE.md
* PLATFORM_CORE_STRUCTURE.md

14.3 O Platform Core **NÃO depende** do domínio de negócio

---

## 15. Decisões Arquiteturais Obrigatórias

* API não usa DTO
* Application usa FluentValidation
* Comunicação via MediatR
* Autorização baseada em permissões
* Auditoria obrigatória

---

## 16. Regra Final

Estas diretrizes são **contratos arquiteturais**.

Qualquer exceção:

* Deve ser consciente
* Deve ser documentada
* Deve ser justificada
