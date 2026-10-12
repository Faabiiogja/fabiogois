# Fabio Gois – AEM (repositório de conteúdo)

Projeto AEM as a Cloud Service usado **apenas como repositório de conteúdo headless**. O front-end é o projeto Next.js [`fabiogois-web`](../fabiogois-web/README.md), editado visualmente pelo **Universal Editor (UE)**. O ambiente local (AEM SDK, Universal Editor Service e proxy HTTPS) está documentado em [`~/Job/aem-sdk/README.md`](../../aem-sdk/README.md).

```
Universal Editor (experience.adobe.com)
        │  iframe                         │ edições
        ▼                                 ▼
 fabiogois-web (Next.js) ── GraphQL ──▶ AEM author/publish (este projeto)
                          persisted queries   Content Fragments
```

## Modelo de conteúdo

| O quê | Onde |
|---|---|
| Modelos de Content Fragment `page`, `hero`, `text-block`, `link`, `header`, `footer` | `ui.content/.../conf/fabiogois/settings/dam/cfm/models` |
| Persisted queries `page-by-slug`, `pages-all`, `global` | `ui.content/.../conf/fabiogois/settings/graphql/persistentQueries` |
| Endpoint GraphQL do projeto | `ui.content/.../content/_cq_graphql/fabiogois/endpoint` |
| CORS (author e publish) | `ui.config/.../osgiconfig/config.author` e `config.publish` (`CORSPolicyImpl~fabiogois`) |
| Conteúdo (fragments) | `/content/dam/fabiogois/site/<página>/` — **criado no AEM, não versionado** |

Header e footer globais ficam em `/content/dam/fabiogois/site/global/` (`header` com o menu, `footer` com links, redes sociais e copyright, e os itens em `links/`), consultados pela query `global`.

Cada página é um fragment `page` (título, slug, descrição) cuja lista `blocks` referencia fragments `hero` / `text-block`, organizados numa pasta por página em `/content/dam/fabiogois/site/`.

Para ver a query respondendo (com o SDK no ar):

```bash
curl -u admin:admin "http://localhost:4502/graphql/execute.json/fabiogois/page-by-slug;slug=home"
```

## Subindo localmente

Pré-requisitos: Java 21 (`.cloudmanager/java-version`), Maven 3.9+, AEM SDK rodando em `localhost:4502` (ver [`aem-sdk/README.md`](../../aem-sdk/README.md)).

1. Build e deploy de tudo:
   ```bash
   mvn clean install -PautoInstallSinglePackage
   ```
   Só conteúdo/configs (mais rápido):
   ```bash
   mvn clean install -pl ui.config,ui.content -PautoInstallPackage
   ```
2. Crie o conteúdo inicial (uma página `home`) em **Assets › Files › fabiogois › site** usando os modelos acima, ou pela API OpenAPI do AEM (`POST /adobe/sites/cf/fragments`). O conteúdo não faz parte do pacote.
3. Siga o README do `fabiogois-web` para subir o front e abrir no Universal Editor.

### Cuidados

- O filtro de `/conf/fabiogois` em `ui.content` está em `mode="merge"`: **alterações em modelos ou persisted queries já existentes não sobrescrevem a versão do AEM** num novo deploy. Localmente, apague o nó no CRXDE (ou crie um modelo novo) antes de reinstalar. Em ambientes compartilhados, prefira editar modelos pela UI do AEM e trazer as mudanças de volta para o repositório.
- Cada persisted query tem o próprio `.content.xml` dentro da sua pasta. Não declare queries inline no `.content.xml` de `persistentQueries`: com `mode="merge"` o pai não é reaplicado e a query nova é criada **sem conteúdo** ("has no Data").
- Campo de referência múltipla a fragments no XML do modelo precisa do nó interno `<field>` com `sling:resourceType`, `name` e `fragmentmodelreference` — sem isso o GraphQL funciona, mas o editor de fragments e a API ignoram o campo.
- Os ajustes de OSGi para o UE local (X-Frame-Options, cookie `SameSite`, `token.required.attr`) **não estão neste repositório de propósito**: só valem para o SDK. Ver `aem-sdk/README.md`.

## Indo para um servidor (AEM as a Cloud Service)

O deploy é feito pelo **Cloud Manager** (pipeline full stack). O que muda em relação ao local:

1. **CORS** – incluir as origens reais do Next nos dois arquivos `CORSPolicyImpl~fabiogois.cfg.json`:
   - author: domínio de preview/edição do Next (o que o UE abre), ex. `https://preview.fabiogois.com`;
   - publish: domínio público, ex. `https://fabiogois.com`.
   Para não fixar domínios no código, use variáveis de ambiente do Cloud Manager: `"alloworigin": ["$[env:NEXT_ORIGIN]"]`. Remova `localhost` do publish.
2. **Dispatcher** (`dispatcher/src`) – os filtros padrão já liberam `/graphql/execute.json*`. Falta:
   - repassar o header `Origin` e devolver os headers `Access-Control-*` do AEM (ou defini-los no vhost) para o domínio do Next;
   - definir cache das persisted queries (TTL via `Cache-Control` configurável por query no AEM) e invalidação;
   - validar com `./bin/validate.sh src` antes do push.
3. **Publicar o conteúdo** – modelos, persisted queries e fragments precisam ser **publicados** (Replicate) para existirem no publish. Persisted queries: publicar pelo console GraphQL; fragments: pela tela de Assets/Content Fragments.
4. **Conteúdo inicial** – como os fragments não estão versionados, recrie-os no ambiente ou exporte um pacote do SDK (Package Manager, filtro `/content/dam/fabiogois/site`) e instale no ambiente de dev.
5. **Universal Editor** – no Cloud **não** há UES local nem proxy HTTPS: a Adobe hospeda o serviço e a autenticação é por IMS. Os ajustes de OSGi do SDK não devem ser levados. Opcional: configurar o *Universal Editor URL Service* (`mappings` como `/content/dam/fabiogois:https://preview.fabiogois.com${path}`) para abrir o UE a partir do console do AEM.
6. **Acesso do Next ao author** – crie uma *Technical Account* (Developer Console › Integrations) e use o token (Bearer) no lugar do `admin:admin` local. O publish serve persisted queries e assets anonimamente.

---

## Documentação original do archetype

This is a project template for AEM-based applications. It is intended as a best-practice set of examples as well as a potential starting point to develop your own functionality.

## Modules

The main parts of the template are:

* [core:](core/README.md) Java bundle containing all core functionality like OSGi services, listeners or schedulers, as well as component-related Java code such as servlets or request filters.
* [it.tests:](it.tests/README.md) Java based integration tests
* [ui.apps:](ui.apps/README.md) contains the /apps (and /etc) parts of the project, ie JS&CSS clientlibs, components, and templates
* [ui.content:](ui.content/README.md) contains sample content using the components from the ui.apps
* ui.config: contains runmode specific OSGi configs for the project
* [ui.frontend:](ui.frontend.general/README.md) an optional dedicated front-end build mechanism (general Webpack project)
* [ui.tests:](ui.tests/README.md) Cypress based UI tests (for other frameworks check [aem-test-samples](https://github.com/adobe/aem-test-samples) repository
* all: a single content package that embeds all of the compiled modules (bundles and content packages) including any vendor dependencies
* analyse: this module runs analysis on the project which provides additional validation for deploying into AEMaaCS

## How to build

To build all the modules run in the project root directory the following command with Maven 3:

    mvn clean install

To build all the modules and deploy the `all` package to a local instance of AEM, run in the project root directory the following command:

    mvn clean install -PautoInstallSinglePackage

Or to deploy it to a publish instance, run

    mvn clean install -PautoInstallSinglePackagePublish

Or alternatively

    mvn clean install -PautoInstallSinglePackage -Daem.port=4503

Or to deploy only the bundle to the author, run

    mvn clean install -PautoInstallBundle

Or to deploy only a single content package, run in the sub-module directory (i.e `ui.apps`)

    mvn clean install -PautoInstallPackage

## Documentation

The build process also generates documentation in the form of README.md files in each module directory for easy reference. Depending on the options you select at build time, the content may be customized to your project.

## Testing

There are three levels of testing contained in the project:

### Unit tests

This show-cases classic unit testing of the code contained in the bundle. To
test, execute:

    mvn clean test

### Integration tests

This allows running integration tests that exercise the capabilities of AEM via
HTTP calls to its API. To run the integration tests, run:

    mvn clean verify -Plocal

Test classes must be saved in the `src/main/java` directory (or any of its
subdirectories), and must be contained in files matching the pattern `*IT.java`.

The configuration provides sensible defaults for a typical local installation of
AEM. If you want to point the integration tests to different AEM author and
publish instances, you can use the following system properties via Maven's `-D`
flag.

| Property              | Description                                         | Default value           |
|-----------------------|-----------------------------------------------------|-------------------------|
| `it.author.url`       | URL of the author instance                          | `http://localhost:4502` |
| `it.author.user`      | Admin user for the author instance                  | `admin`                 |
| `it.author.password`  | Password of the admin user for the author instance  | `admin`                 |
| `it.publish.url`      | URL of the publish instance                         | `http://localhost:4503` |
| `it.publish.user`     | Admin user for the publish instance                 | `admin`                 |
| `it.publish.password` | Password of the admin user for the publish instance | `admin`                 |

The integration tests in this archetype use the [AEM Testing
Clients](https://github.com/adobe/aem-testing-clients) and showcase some
recommended [best
practices](https://github.com/adobe/aem-testing-clients/wiki/Best-practices) to
be put in use when writing integration tests for AEM.

## Static Analysis

The `analyse` module performs static analysis on the project for deploying into AEMaaCS. It is automatically
run when executing

    mvn clean install

from the project root directory. Additional information about this analysis and how to further configure it
can be found here https://github.com/adobe/aemanalyser-maven-plugin

### UI tests

They will test the UI layer of your AEM application using Cypress framework.

Check README file in `ui.tests` module for more details.

Examples of UI tests in different frameworks can be found here: https://github.com/adobe/aem-test-samples

## ClientLibs

The frontend module is made available using an [AEM ClientLib](https://helpx.adobe.com/experience-manager/6-5/sites/developing/using/clientlibs.html). When executing the NPM build script, the app is built and the [`aem-clientlib-generator`](https://github.com/wcm-io-frontend/aem-clientlib-generator) package takes the resulting build output and transforms it into such a ClientLib.

A ClientLib will consist of the following files and directories:

- `css/`: CSS files which can be requested in the HTML
- `css.txt` (tells AEM the order and names of files in `css/` so they can be merged)
- `js/`: JavaScript files which can be requested in the HTML
- `js.txt` (tells AEM the order and names of files in `js/` so they can be merged
- `resources/`: Source maps, non-entrypoint code chunks (resulting from code splitting), static assets (e.g. icons), etc.

## Simplified Index Management

If you're building for AEM as a Cloud Service, this project includes a `diff.index` under
`ui.apps/src/main/content/jcr_root/_oak_index` ([documentation](https://oak-indexing.github.io/oakTools/simplified.html)).
Packaging it triggers a benign Maven build warning from the `jackrabbit-nodetypes` validator
(`... is not allowed as child of node with potential default types [nt:folder] ... /oak:index/diff.index`),
because `/oak:index`'s real node type isn't declared in any content package. It's a known false
positive and safe to ignore.

## Maven settings

The project comes with the auto-public repository configured. To setup the repository in your Maven settings, refer to:

    http://helpx.adobe.com/experience-manager/kb/SetUpTheAdobeMavenRepository.html
