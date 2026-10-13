# platform-gitops

**Estado desejado** do AKS da plataforma de dados event-driven. O Git é a fonte da verdade: o **Flux v2**, instalado como extensão gerenciada do AKS pelo `platform-infra`, reconcilia este repositório continuamente.

> Nenhum pipeline tem credencial do cluster. Para mudar algo no AKS, abra um PR aqui (ou deixe o `cd-gitops-promote` abrir por você).

## Estrutura

```
clusters/
  dev/
    infrastructure/   # raiz da Kustomization Flux "infrastructure"
    domains/          # raiz da Kustomization Flux "domains" (dependsOn: infrastructure)
infrastructure/
  base/               # componentes compartilhados (hoje: demo-podinfo)
  overlays/dev/       # ajustes por ambiente
domains/
  <dominio>/
    base/             # Namespace, ServiceAccount (Workload Identity), HelmRelease(s)
    overlays/<env>/   # versão da imagem e valores por ambiente (alterado pelo CD)
```

As Kustomizations do Flux **não** estão neste repositório: elas são declaradas no recurso `azurerm_kubernetes_flux_configuration` do `platform-infra` (stack `domains`). O Terraform também injeta, via `postBuild.substitute`, as variáveis `${...}` usadas aqui:

| Variável | Origem |
|---|---|
| `AZURE_TENANT_ID` | Tenant do Entra ID |
| `EVENTHUB_NAMESPACE_FQDN` | Namespace Event Hubs (`evhns-edp-dev.servicebus.windows.net`) |
| `ACR_LOGIN_SERVER` | ACR da plataforma |
| `<DOMINIO>_<WORKLOAD>_CLIENT_ID` | Managed Identity da workload (ex.: `PEDIDOS_PEDIDOS_API_CLIENT_ID`) |

## Fluxos

| Ação | Como |
|---|---|
| Deploy de nova versão em dev | O CI do `<dominio>-service` faz commit direto no overlay `dev` (`cd-gitops-promote`, `mode=commit`) |
| Promoção para prd | PR gerado pelo `cd-gitops-promote` (`mode=pull-request`); a aprovação é a promoção |
| Rollback | `git revert` do commit de promoção |
| Onboarding de domínio | PR adicionando `domains/<dominio>/` + a entrada em `clusters/<env>/domains/kustomization.yaml` |
| Drift (alguém mudou no cluster) | O Flux desfaz no próximo ciclo de reconciliação |

## Validação

O CI (`ci-kustomize@v1`) roda `kustomize build` em cada raiz e `kubeconform` (com os CRDs do Flux). Para rodar localmente:

```bash
kubectl kustomize clusters/dev/infrastructure
```

## ADRs relacionados

ADR-0004 (Bounded Context como unidade de provisionamento) e ADR-0007 (Polyrepo + GitOps com Flux v2).
