# Guia de Arquitetura — Plataforma White Label Multiplataforma

Oct 8, 2026 · @Angelo silveira

## 1. Contexto e problema

A solução é tratar o tema como **dados versionados (design tokens)**, nunca como código React: o Studio edita um JSON neutro, um pipeline o traduz para cada plataforma e SDKs nativos o aplicam em runtime.

**Objetivo.** Construir duas aplicações:

1. **White Label Studio** (React): painel onde o cliente escolhe cores, tipografia, raios, espaçamentos, logos e algumas variações de layout (ex.: checkout).
2. **Aplicações consumidoras**: o app do cliente (iOS, Android, React Native, Flutter, Web) e o nosso checkout, que leem o tema publicado e se redesenham.

**O problema central.** O Studio é React, mas os apps não são. Um botão em SwiftUI não entende CSS, e um `Button` do Jetpack Compose não lê um objeto JavaScript. Portanto o que trafega entre Studio e apps não pode ser componente nem CSS: precisa ser um contrato de dados que todas as linguagens leiam.

**Princípios que guiam o resto do guia:**

- **Contrato único, renderização nativa.** Um JSON de tokens (padrão W3C DTCG) é a fonte da verdade; cada plataforma renderiza com seus componentes nativos.
- **Componentes conhecem papéis, não cores.** O botão usa `color.action.primary.background`, nunca `#6A1BFF`. Trocar o tema = trocar o valor do papel.
- **Runtime por padrão, build-time como fallback.** Mudar o tema não pode exigir nova versão na App Store/Play Store.
- **Nunca quebrar o app.** Sempre existe um tema padrão embarcado e um último tema válido em cache.
- **Customização com trilhos.** O cliente escolhe dentro de limites (contraste, tamanhos, layouts pré-aprovados) para não destruir acessibilidade nem conversão.

## 2. Visão geral da arquitetura

O sistema tem quatro blocos: o Studio edita, a Theme API versiona, o build worker traduz e a CDN distribui para SDKs leves em cada plataforma.

&#91;embedded content: arquitetura · Studio, API, pipeline, CDN e 5 SDKs\]

A CDN é o ponto de contrato: o Studio pode mudar de tecnologia sem afetar nenhum app, e um app novo só precisa de um SDK que leia o manifest.

## 3. Modelo de design tokens

Os tokens ficam em três camadas; o cliente só edita a primeira e parte da segunda, e os componentes só leem a segunda e a terceira.

| Camada | Exemplo | Quem define | Quem consome |
| --- | --- | --- | --- |
| Primitivo (reference) | `palette.brand.500 = #6A1BFF`, `radius.md = 8` | Cliente, via Studio | Apenas tokens semânticos |
| Semântico (system) | `color.action.primary.background → {palette.brand.500}` | Nosso time (mapa padrão), cliente pode sobrescrever | Componentes |
| Componente | `button.primary.radius → {radius.md}` | Nosso time; cliente ajusta poucos | Componente específico |

**Formato.** JSON no padrão [W3C Design Tokens (DTCG)](https://www.designtokens.org/), com `$value`, `$type` e referências `{...}`. Ele é neutro de linguagem e é lido pelo Style Dictionary, Tokens Studio e Figma.

```json
{
  "$schema": "https://wl.suaempresa.com/schema/theme/v1.json",
  "meta": { "tenantId": "acme", "version": 14, "schemaVersion": "1.2.0" },
  "palette": {
    "brand": { "500": { "$type": "color", "$value": "#6A1BFF" } },
    "neutral": { "0": { "$type": "color", "$value": "#FFFFFF" } }
  },
  "color": {
    "action": { "primary": {
      "background": { "$type": "color", "$value": "{palette.brand.500}" },
      "foreground": { "$type": "color", "$value": "{palette.neutral.0}" }
    } }
  },
  "radius": { "md": { "$type": "dimension", "$value": "8px" } },
  "typography": { "body": { "$type": "typography", "$value": {
    "fontFamily": "Inter", "fontSize": "16px", "fontWeight": 400, "lineHeight": 1.5 } } },
  "modes": { "dark": { "color.action.primary.background": "{palette.brand.300}" } }
}
```

**Regras do contrato:**

- **Nomes semânticos são a API pública.** Renomear um token semântico é breaking change; adicionar é minor.
- **Dois versionamentos distintos.** `schemaVersion` (SemVer, versão do contrato, controla compatibilidade com SDKs) e `version` (inteiro, cada publicação do tema do cliente).
- **Unidades neutras.** Dimensões em `px` lógicos no JSON; o pipeline converte para `pt` (iOS), `dp`/`sp` (Android) e `rem` (Web).
- **Modos.** Light/dark e, se necessário, alto contraste como overrides da camada semântica, não temas inteiros duplicados.
- **Assets por referência.** Logos e ícones entram como URL assinada + hash, nunca base64 dentro do JSON.
- **Fontes.** Lista fechada de famílias homologadas (licença e disponibilidade nas 3 plataformas); fonte custom só em plano enterprise, com download validado.

## 4. Aplicação 1: White Label Studio (React)

O Studio é um editor de tema com preview ao vivo em todas as plataformas; ele nunca gera código, só edita e publica o JSON de tokens.

**Módulos principais:**

1. **Marca:** logo (claro/escuro), favicon, ícone do app, cor primária e secundária.
2. **Cores:** a partir de 1–2 cores-semente, gerar a escala 50–900 automaticamente (OKLCH) e sugerir o mapa semântico. Edição manual avançada atrás de um toggle "modo avançado".
3. **Tipografia:** família (lista homologada), escala de tamanhos, peso de títulos.
4. **Forma:** raio (reto / suave / arredondado / pílula), densidade (compacto / confortável), elevação.
5. **Componentes:** botões, inputs, cards, badges, com estados (default, hover, pressed, disabled, focus).
6. **Layouts:** variações pré-aprovadas de telas-chave, como o checkout (ver seção 9).
7. **Publicação:** diff com a versão atual, validação, agendamento, histórico e rollback em um clique.

**Decisões de UX:**

- **Preview multiplataforma lado a lado.** Molduras de iPhone, Android e desktop renderizando o mesmo tema. Web com CSS variables; mobile com React Native Web ou réplicas fiéis dos componentes nativos. Para fidelidade total, oferecer QR code que abre o tema em rascunho no app de preview nativo.
- **Comece pelo simples.** Fluxo guiado de 3 passos (logo → cor → estilo) gera um tema completo em menos de 2 minutos; o editor detalhado vem depois.
- **Validação em tempo real, não no fim.** Contraste WCAG AA (4.5:1 texto, 3:1 componentes) verificado a cada mudança, com sugestão automática do tom acessível mais próximo.
- **Rascunho vs publicado sempre visível.** Badge de estado no topo e diff antes de publicar; publicar afeta usuários finais em produção.
- **Desfazer barato.** Histórico de versões com preview de qualquer versão e rollback.
- **Papéis e permissões.** Designer edita, aprovador publica (fluxo opcional de aprovação para clientes enterprise).

**Stack sugerida:** React + TypeScript, Vite ou Next.js, Zustand para o estado do editor, Zod (gerado a partir do JSON Schema) para validação, `culori` para cores em OKLCH e contraste, TanStack Query para API, React Native Web para os previews mobile.

## 5. Backend de temas: API, publicação e CDN

Os apps nunca falam com o banco nem com a API de edição; eles leem arquivos imutáveis e cacheáveis numa CDN, o que dá latência baixa e isola o Studio do tráfego de produção.

**Componentes:**

- **Theme API (escrita, autenticada):** CRUD de rascunhos, validação, publicação, rollback. Usada só pelo Studio.
- **Banco:** PostgreSQL com o tema em JSONB, tabelas `tenants`, `themes`, `theme_versions` (imutáveis), `audit_log`.
- **Build worker:** ao publicar, uma fila (SQS, Pub/Sub ou BullMQ) dispara o pipeline da seção 6, que gera os artefatos por plataforma.
- **Storage + CDN:** S3/GCS atrás de CloudFront/Cloudflare, com arquivos versionados e imutáveis.
- **Manifest (leitura, pública ou com chave do app):** um arquivo pequeno que aponta para a versão vigente.

**Estrutura de URLs:**

| Arquivo | Cache | Conteúdo |
| --- | --- | --- |
| `/t/{tenant}/manifest.json` | 60 s + ETag | Versão atual, `schemaVersion`, hash e URLs dos artefatos |
| `/t/{tenant}/v{n}/tokens.json` | 1 ano, imutável | JSON resolvido (sem aliases), usado por mobile em runtime |
| `/t/{tenant}/v{n}/theme.css` | 1 ano, imutável | CSS variables para Web e checkout |
| `/t/{tenant}/v{n}/assets/*` | 1 ano, imutável | Logos e ícones com hash no nome |

**Fluxo de leitura no app:** busca o manifest (requisição leve, normalmente 304), compara o hash com o cache local e só baixa `tokens.json` quando mudou.

**Atualização quase em tempo real (opcional):** push silencioso (APNs/FCM) ou SSE/WebSocket avisando "nova versão" para apps abertos; caso contrário, a checagem ocorre no próximo foreground.

**Contrato do manifest:**

```json
{
  "tenantId": "acme",
  "version": 14,
  "schemaVersion": "1.2.0",
  "minSdk": { "ios": "2.0.0", "android": "2.0.0", "web": "2.0.0" },
  "hash": "sha256-9f2c…",
  "tokens": "https://cdn.wl.com/t/acme/v14/tokens.json",
  "css": "https://cdn.wl.com/t/acme/v14/theme.css",
  "publishedAt": "2026-10-08T12:00:00Z"
}
```

## 6. Pipeline de transformação (Style Dictionary)

Um único pipeline, rodando no build worker, transforma o JSON DTCG em um artefato por plataforma; é ele que resolve a "tradução" entre React e nativo.

**Etapas ao publicar:**

1. Validar contra o JSON Schema da `schemaVersion`.
2. Resolver aliases (`{palette.brand.500}` vira `#6A1BFF`) e aplicar modos.
3. Rodar checagens: contraste, tokens obrigatórios presentes, assets acessíveis.
4. Gerar artefatos com [Style Dictionary](https://styledictionary.com/) (v4+, suporta DTCG nativamente).
5. Calcular hash, subir para storage, atualizar manifest, invalidar só o manifest na CDN.

**Saídas por plataforma:**

| Plataforma | Artefato runtime | Artefato build-time (fallback embarcado) | Conversões |
| --- | --- | --- | --- |
| Web / checkout | `theme.css` (CSS variables) | Pacote npm `@wl/tokens` (CSS + TS) | px → rem, cores em hex/rgb |
| React Native | `tokens.json` resolvido | Pacote npm com `theme.ts` tipado | px → número sem unidade |
| iOS | `tokens.json` resolvido | `Theme+Generated.swift` + Asset Catalog | px → pt, hex → `Color`/`UIColor` |
| Android | `tokens.json` resolvido | `Tokens.kt` (Compose) + `colors.xml`/`dimens.xml` | px → dp/sp, hex → ARGB |
| Flutter | `tokens.json` resolvido | `tokens.dart` (`ThemeExtension`) | px → lógico, hex → `Color(0xFF…)` |

**Por que dois artefatos:** o runtime permite trocar o tema sem release; o build-time garante que o app abre com uma identidade correta mesmo offline no primeiro uso e dá tipagem forte aos desenvolvedores (`theme.color.action.primary.background` com autocomplete).

**Geração de tipos:** o mesmo pipeline gera as interfaces (TS, Swift `struct`, Kotlin `data class`, Dart `class`) a partir do schema, publicadas como parte de cada SDK. Assim, um token novo aparece em todas as plataformas ao mesmo tempo.

## 7. Consumo por plataforma

Cada plataforma recebe um **SDK de tema** com a mesma API conceitual: `init(tenantId, apiKey)`, `theme` observável, `refresh()` e componentes base que já leem os tokens.

### 7.1 Web e checkout (React ou qualquer framework)

CSS variables são o mecanismo: o componente referencia a variável, e trocar o tema é trocar um arquivo CSS, sem re-render do React.

```css
/* theme.css gerado */
:root {
  --wl-color-action-primary-bg: #6A1BFF;
  --wl-color-action-primary-fg: #FFFFFF;
  --wl-radius-md: 0.5rem;
}
[data-theme="dark"] { --wl-color-action-primary-bg: #9C6BFF; }
```

```tsx
// SDK web
await WhiteLabel.init({ tenantId: 'acme' }); // injeta <link href=".../v14/theme.css">

.btn-primary {
  background: var(--wl-color-action-primary-bg);
  color: var(--wl-color-action-primary-fg);
  border-radius: var(--wl-radius-md);
}
```

Para SSR (Next.js), o servidor resolve o tenant (subdomínio ou header) e inclui o `<link>` no HTML, evitando o "flash" de tema padrão. Tailwind pode mapear `colors.primary` para `var(--wl-...)`.

### 7.2 React Native

```tsx
<ThemeProvider tenantId="acme" fallback={embeddedTheme}>
  <App />
</ThemeProvider>

const { color, radius } = useTheme();
<Pressable style={{ backgroundColor: color.action.primary.background,
                    borderRadius: radius.md }} />
```

O provider carrega do cache (MMKV) de forma síncrona, renderiza, e em paralelo busca o manifest; se houver nova versão, atualiza o contexto.

### 7.3 iOS (SwiftUI e UIKit)

```swift
// Swift Package: WhiteLabelKit
@main struct MyApp: App {
  @StateObject var theme = ThemeStore(tenantId: "acme", fallback: .embedded)
  var body: some Scene {
    WindowGroup { RootView().environmentObject(theme) }
  }
}

struct PrimaryButton: View {
  @EnvironmentObject var theme: ThemeStore
  var body: some View {
    Button("Pagar") { }
      .background(theme.color.action.primary.background)
      .clipShape(RoundedRectangle(cornerRadius: theme.radius.md))
  }
}
```

`ThemeStore` é um `ObservableObject` (ou `@Observable` no iOS 17+) que decodifica o JSON via `Codable` e converte hex em `Color`. Em UIKit, o SDK publica um `NotificationCenter`/Combine e os componentes base reaplicam estilo; `UIAppearance` cobre barras e controles do sistema.

### 7.4 Android (Jetpack Compose e Views)

```kotlin
// Gradle: com.suaempresa:whitelabel-compose
setContent {
  val theme by WhiteLabel.theme.collectAsState(initial = EmbeddedTheme)
  WhiteLabelTheme(theme) { App() }
}

@Composable fun PrimaryButton(onClick: () -> Unit) {
  val t = LocalWLTheme.current
  Button(onClick, shape = RoundedCornerShape(t.radius.md.dp),
    colors = ButtonDefaults.buttonColors(containerColor = t.color.action.primary.background))
  { Text("Pagar") }
}
```

`WhiteLabelTheme` também mapeia tokens para o `MaterialTheme` (`colorScheme`, `shapes`, `typography`), então componentes Material já saem com a marca. Em Views/XML, cores de `colors.xml` não mudam em runtime: o SDK oferece componentes custom (`WLButton`) que leem o tema em código.

### 7.5 Flutter

O SDK expõe um `ThemeExtension<WLTokens>` e preenche o `ThemeData`; `Theme.of(context).extension<WLTokens>()` dá acesso aos tokens.

### 7.6 Checkout dentro do app (WebView)

Se o checkout for web embutido no app, o app passa `tenantId`, modo (claro/escuro) e versão do tema na URL ou via bridge JS. O checkout carrega o mesmo `theme.css` e fica visualmente idêntico às telas nativas.

**Resumo:** o JSON é o mesmo para todos; muda só o "adaptador" (CSS variables, Context, EnvironmentObject, CompositionLocal, ThemeExtension) que cada plataforma usa nativamente.

### 7.7 O que é um SDK aqui e como reduzir quantos construir

**SDK é uma biblioteca, não uma aplicação.** Só existe uma aplicação de edição: o White Label Studio. O SDK é um pacote que o desenvolvedor do cliente instala no app dele (npm, Swift Package, Gradle) e que não tem tela de edição. O fluxo é de mão única: o Studio escreve, os SDKs só leem.

Cada SDK faz três coisas: busca o tema na CDN (manifest + `tokens.json`), mantém cache e fallback, e entrega os tokens aos componentes da plataforma.

**Por que um por tecnologia:** cada runtime aplica estilo de um jeito (CSS variables, Context do React Native, `EnvironmentObject` no SwiftUI, `CompositionLocal` no Compose). Não há como um único pacote cobrir todas.

**Como reduzir o esforço:**

| Estratégia | Efeito | Quando usar |
| --- | --- | --- |
| Priorizar por cliente | Construir só os SDKs que a base atual usa | Sempre: ordem definida pela proporção RN/nativo/Flutter |
| React Native primeiro | Um SDK em TypeScript cobre iOS e Android | Se a maioria dos apps clientes for RN |
| Web mínimo | `theme.css` + script de \~2 KB que injeta o `<link>` e troca o modo | Web e checkout |
| Núcleo nativo compartilhado (Kotlin Multiplatform) | Manifest, validação, cache e fallback escritos uma vez para iOS e Android; cada plataforma só ganha a camada fina de UI | Quando os SDKs iOS e Android forem necessários |
| Sem SDK (contrato aberto) | Publicar `tokens.json` documentado e deixar o cliente integrar | Clientes com time próprio; perde cache, fallback e componentes prontos |

### 7.8 Exemplo: SDK React Native

Para o cliente, a integração são dois passos: instalar o pacote e envolver o app no `WhiteLabelProvider` com o `tenantId` e a chave gerada no Studio. Todo o resto (download, cache, fallback, modo escuro, atualização) fica dentro do SDK.

**Lado do cliente (app que consome)**

```bash
npm install @suaempresa/whitelabel-react-native react-native-mmkv
```

```tsx
// App.tsx do cliente
import { WhiteLabelProvider } from '@suaempresa/whitelabel-react-native';

export default function App() {
  return (
    <WhiteLabelProvider tenantId="acme" apiKey="pk_live_xxx">
      <Navigation />
    </WhiteLabelProvider>
  );
}

// Qualquer tela: componentes prontos do SDK ou o hook
import { WLButton, useTheme } from '@suaempresa/whitelabel-react-native';

function CheckoutScreen() {
  const { theme } = useTheme();
  return (
    <View style={{ backgroundColor: theme.color.surface.default, padding: theme.spacing.md }}>
      <WLButton title="Pagar" onPress={pay} />
    </View>
  );
}
```

**Estrutura do pacote (nosso lado)**

```text
whitelabel-react-native/
├─ src/
│  ├─ index.ts            # API pública
│  ├─ types.ts            # gerado pelo pipeline a partir do schema
│  ├─ defaultTheme.ts     # tema embarcado (fallback)
│  ├─ client.ts           # manifest, download, validação
│  ├─ storage.ts          # cache em MMKV
│  ├─ ThemeProvider.tsx   # Context + ciclo de atualização
│  ├─ hooks.ts            # useTheme, useThemedStyles
│  └─ components/WLButton.tsx
└─ package.json           # peerDeps: react, react-native, react-native-mmkv
```

**`types.ts`** (gerado; trecho)

```ts
export interface WLTheme {
  meta: { tenantId: string; version: number; schemaVersion: string };
  color: {
    action: { primary: { background: string; foreground: string; pressed: string } };
    surface: { default: string };
    text: { primary: string; secondary: string };
  };
  radius: { sm: number; md: number; lg: number };
  spacing: { xs: number; sm: number; md: number; lg: number };
  typography: { body: { fontFamily: string; fontSize: number; fontWeight: '400' | '500' | '600' | '700'; lineHeight: number } };
  modes?: { dark?: DeepPartial<Pick<WLTheme, 'color'>> };
}
export type DeepPartial<T> = { [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K] };
```

**`client.ts`**

```ts
const CDN = 'https://cdn.wl.com';
const SUPPORTED_SCHEMA_MAJOR = 1;

export interface Manifest { version: number; schemaVersion: string; hash: string; tokens: string; applyPolicy?: 'immediate' | 'nextLaunch' }

export async function fetchManifest(tenantId: string, apiKey: string, etag?: string) {
  const res = await fetch(`${CDN}/t/${tenantId}/manifest.json`, {
    headers: { 'x-wl-key': apiKey, ...(etag ? { 'If-None-Match': etag } : {}) },
  });
  if (res.status === 304) return { changed: false as const };
  if (!res.ok) throw new Error(`manifest ${res.status}`);
  const manifest: Manifest = await res.json();
  return { changed: true as const, manifest, etag: res.headers.get('ETag') ?? undefined };
}

export async function fetchTheme(manifest: Manifest): Promise<WLTheme> {
  const major = Number(manifest.schemaVersion.split('.')[0]);
  if (major !== SUPPORTED_SCHEMA_MAJOR) throw new Error('schema incompatível');
  const res = await fetch(manifest.tokens);
  const theme = await res.json();
  return validateTheme(theme); // zod: rejeita tema inválido, preenche ausentes com default
}
```

**`storage.ts`**

```ts
import { MMKV } from 'react-native-mmkv';
const store = new MMKV({ id: 'wl-theme' });

export const loadCached = (tenantId: string): WLTheme | null => {
  const raw = store.getString(`theme:${tenantId}`);
  return raw ? JSON.parse(raw) : null; // síncrono: sem flash de tema padrão
};
export const saveCached = (tenantId: string, theme: WLTheme, etag?: string) => {
  store.set(`theme:${tenantId}`, JSON.stringify(theme));
  if (etag) store.set(`etag:${tenantId}`, etag);
};
export const loadEtag = (tenantId: string) => store.getString(`etag:${tenantId}`);
```

**`ThemeProvider.tsx`**

```tsx
import React, { createContext, useCallback, useEffect, useMemo, useState } from 'react';
import { AppState, useColorScheme } from 'react-native';

type Status = 'cached' | 'fallback' | 'updated' | 'error';
export interface ThemeContextValue { theme: WLTheme; mode: 'light' | 'dark'; status: Status; refresh: () => Promise<void> }
export const ThemeContext = createContext<ThemeContextValue | null>(null);

interface Props {
  tenantId: string;
  apiKey: string;
  fallback?: WLTheme;
  mode?: 'system' | 'light' | 'dark';
  children: React.ReactNode;
}

export function WhiteLabelProvider({ tenantId, apiKey, fallback = defaultTheme, mode = 'system', children }: Props) {
  // 1. Cache síncrono ou tema embarcado: o app sempre abre com uma marca válida
  const [base, setBase] = useState<WLTheme>(() => loadCached(tenantId) ?? fallback);
  const [status, setStatus] = useState<Status>(() => (loadCached(tenantId) ? 'cached' : 'fallback'));
  const system = useColorScheme();
  const resolvedMode = mode === 'system' ? (system ?? 'light') : mode;

  // 2. Atualização em background; qualquer falha mantém o tema atual
  const refresh = useCallback(async () => {
    try {
      const result = await fetchManifest(tenantId, apiKey, loadEtag(tenantId));
      if (!result.changed) return;
      const next = await fetchTheme(result.manifest);
      saveCached(tenantId, next, result.etag);
      if (result.manifest.applyPolicy !== 'nextLaunch') {
        setBase(next);
        setStatus('updated');
      }
    } catch (e) {
      setStatus((s) => (s === 'fallback' ? 'error' : s));
      // reportar telemetria aqui
    }
  }, [tenantId, apiKey]);

  // 3. Checa ao abrir e sempre que o app volta para o primeiro plano
  useEffect(() => {
    refresh();
    const sub = AppState.addEventListener('change', (s) => s === 'active' && refresh());
    return () => sub.remove();
  }, [refresh]);

  // 4. Aplica o modo escuro como override da camada semântica
  const theme = useMemo(
    () => (resolvedMode === 'dark' && base.modes?.dark ? deepMerge(base, base.modes.dark) : base),
    [base, resolvedMode],
  );

  const value = useMemo(() => ({ theme, mode: resolvedMode, status, refresh }), [theme, resolvedMode, status, refresh]);
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}
```

**`hooks.ts`**

```ts
export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme precisa estar dentro de <WhiteLabelProvider>');
  return ctx;
}

// Evita recriar estilos a cada render: só recalcula quando o tema muda
export function useThemedStyles<T extends StyleSheet.NamedStyles<T>>(factory: (t: WLTheme) => T) {
  const { theme } = useTheme();
  return useMemo(() => StyleSheet.create(factory(theme)), [theme]);
}
```

**`components/WLButton.tsx`**

```tsx
export function WLButton({ title, onPress, disabled }: { title: string; onPress: () => void; disabled?: boolean }) {
  const s = useThemedStyles((t) => ({
    base: { minHeight: 48, paddingHorizontal: t.spacing.lg, borderRadius: t.radius.md,
            alignItems: 'center', justifyContent: 'center', backgroundColor: t.color.action.primary.background },
    pressed: { backgroundColor: t.color.action.primary.pressed },
    label: { color: t.color.action.primary.foreground, fontFamily: t.typography.body.fontFamily,
             fontSize: t.typography.body.fontSize, fontWeight: '600' },
  }));
  return (
    <Pressable accessibilityRole="button" disabled={disabled} onPress={onPress}
      style={({ pressed }) => [s.base, pressed && s.pressed, disabled && { opacity: 0.4 }]}>
      <Text style={s.label}>{title}</Text>
    </Pressable>
  );
}
```

**Pontos de atenção**

- **Navegação e barras do sistema:** o SDK exporta `toNavigationTheme(theme)` para o React Navigation e ajusta `StatusBar` conforme o contraste da cor de fundo.
- **Fontes:** o RN só usa fontes empacotadas no app. O SDK verifica se a família do tema está disponível e, se não, usa a do sistema; por isso a lista de fontes homologadas (seção 3).
- **Componentes do próprio cliente:** ele pode usar `useTheme()` ou `useThemedStyles()` nos componentes dele, sem depender só dos nossos.
- **Expo:** funciona igual; com Expo Go, trocar MMKV por `expo-secure-store` ou AsyncStorage (perde a leitura síncrona).

### 7.9 Quando o cliente não usa nossos componentes

Os componentes prontos (`WLButton` e afins) são opcionais; o contrato obrigatório é só o provider, que entrega os tokens. O cliente integra por um destes três caminhos, sem trocar a biblioteca de UI que já usa.

| Situação do cliente | Caminho | O que muda no código dele |
| --- | --- | --- |
| Componentes próprios | Hook `useTheme()` / `useThemedStyles()` | Valores fixos de estilo viram tokens |
| Biblioteca de UI (Paper, Tamagui, NativeBase, styled-components) | Adaptador do SDK gera o tema da biblioteca | Só envolve o provider da biblioteca; nenhum componente muda |
| Design system interno com tema próprio | Mapa de tokens documentado + adaptador escrito pelo cliente | Uma função de mapeamento |

**Caminho 1: componentes próprios**

```tsx
function MeuBotao({ title, onPress }) {
  const s = useThemedStyles((t) => ({
    btn: { backgroundColor: t.color.action.primary.background, borderRadius: t.radius.md },
    label: { color: t.color.action.primary.foreground },
  }));
  return (
    <TouchableOpacity style={s.btn} onPress={onPress}>
      <Text style={s.label}>{title}</Text>
    </TouchableOpacity>
  );
}
```

**Caminho 2: adaptador para biblioteca de UI** (exemplo com React Native Paper)

```tsx
import { useTheme, toPaperTheme } from '@suaempresa/whitelabel-react-native';
import { PaperProvider } from 'react-native-paper';

function ThemeBridge({ children }: { children: React.ReactNode }) {
  const { theme, mode } = useTheme();
  const paperTheme = useMemo(() => toPaperTheme(theme, mode), [theme, mode]);
  return <PaperProvider theme={paperTheme}>{children}</PaperProvider>;
}

export default function App() {
  return (
    <WhiteLabelProvider tenantId="acme" apiKey="pk_live_xxx">
      <ThemeBridge>
        <Navigation />
      </ThemeBridge>
    </WhiteLabelProvider>
  );
}
```

O adaptador é apenas um mapa de tokens para o formato da biblioteca:

```ts
import { MD3LightTheme, MD3DarkTheme } from 'react-native-paper';

export function toPaperTheme(t: WLTheme, mode: 'light' | 'dark') {
  const base = mode === 'dark' ? MD3DarkTheme : MD3LightTheme;
  return {
    ...base,
    roundness: t.radius.md,
    colors: {
      ...base.colors,
      primary: t.color.action.primary.background,
      onPrimary: t.color.action.primary.foreground,
      background: t.color.surface.default,
      onSurface: t.color.text.primary,
    },
  };
}
```

**Caminho 3: design system interno.** Publicamos a tabela oficial "token semântico → papel" (ex.: `color.action.primary.background` = cor de fundo da ação principal) e o cliente escreve a própria função de mapeamento, no mesmo formato do `toPaperTheme`.

**Decisões de empacotamento**

- **Dividir em dois pacotes:** `whitelabel-react-native` (provider, hooks, adaptadores, zero componentes visuais) e `whitelabel-react-native-ui` (componentes prontos, opcional). Quem tem design system próprio não carrega UI que não usa.
- **Adaptadores como entry points separados** (`@suaempresa/whitelabel-react-native/paper`, `/tamagui`), com a biblioteca como `peerDependency` opcional, para não forçar dependências.
- **Adaptadores oficiais só para as 2 ou 3 bibliotecas mais usadas pela base**; as demais seguem o caminho 3.
- **Mesmo princípio no nativo:** no Android o adaptador preenche o `MaterialTheme`; no iOS, o `UIAppearance` ou o tema do design system do cliente; no Flutter, o `ThemeData`.

## 8. Runtime vs build-time, cache e fallback

A recomendação é híbrida: tema embarcado no build como piso de segurança, e tema remoto em runtime como fonte normal.

| Critério | Runtime (remoto) | Build-time (embarcado) |
| --- | --- | --- |
| Mudar tema sem release | Sim | Não, exige nova versão na loja |
| Funciona offline no 1º uso | Não | Sim |
| Ícone do app e splash nativa | Não é possível | Sim (único jeito) |
| Tipagem forte para devs | Via tipos gerados | Sim |

**Ordem de resolução ao abrir o app:**

1. Ler o último tema válido do cache local (síncrono, < 10 ms) e renderizar.
2. Sem cache: usar o tema embarcado.
3. Em background: buscar o manifest; se o hash mudou e `minSdk` é compatível, baixar, validar o schema e trocar.
4. Qualquer falha (rede, JSON inválido, schema incompatível) mantém o tema atual e reporta telemetria.

**Detalhes que evitam bugs:**

- **Troca sem "piscada".** Aplicar o novo tema no próximo cold start ou numa transição de tela; trocar no meio de um fluxo de pagamento confunde o usuário. Flag `applyPolicy: immediate | nextLaunch` no manifest.
- **Compatibilidade.** SDK antigo ignora tokens que não conhece e usa default para tokens obrigatórios ausentes.
- **Cache:** iOS em `Application Support`, Android em DataStore, RN em MMKV, Web em `localStorage` + Service Worker.
- **Itens que só mudam via loja:** ícone do app, splash nativa, nome do app. O Studio deve deixar isso claro e, se houver pipeline de apps white label, disparar o build (Fastlane + CI).

## 9. Layouts configuráveis (server-driven UI controlado)

Para layout, o cliente escolhe entre **variantes pré-construídas** e configura flags; não montamos telas livres, porque isso explode a complexidade nativa e o risco de quebrar a conversão do checkout.

**Como funciona:** cada tela configurável tem um contrato de layout ao lado dos tokens.

```json
"layouts": {
  "checkout": {
    "variant": "one-page",          // "one-page" | "steps" | "compact"
    "summaryPosition": "top",       // "top" | "bottom" | "side" (web)
    "paymentMethodsOrder": ["pix", "card", "boleto"],
    "showTrustBadges": true,
    "header": { "logo": true, "backButton": true }
  }
}
```

Cada SDK implementa as variantes com componentes nativos e escolhe por `variant`. Valor desconhecido cai no default.

**Níveis de maturidade:**

1. **Tokens** (cores, tipografia, forma): todas as telas.
2. **Variantes e flags** (este modelo): telas críticas como checkout, login, home.
3. **Composição de seções** (lista ordenada de blocos de um catálogo fechado): só se houver demanda real, por exemplo a home. Avaliar ferramentas como Airbnb Ghost Platform ou Shopify-like schema antes de construir do zero.

**Regra de UX:** toda variante passa por teste de usabilidade e, de preferência, teste A/B antes de entrar no catálogo; o Studio mostra o impacto esperado (ex.: "one-page converte melhor em mobile") para guiar a escolha.

## 10. Segurança, acessibilidade e governança

O tema é conteúdo vindo de fora que roda em produção nos apps, então é validado como entrada não confiável em toda a cadeia.

**Segurança**

- Valores de token validados por tipo (regex de cor, faixa de dimensão); nada de CSS livre, para evitar injeção de `url()` ou `expression` no `theme.css`.
- Assets: upload com verificação de MIME, limite de tamanho, SVG sanitizado (sem scripts) ou convertido para PNG/PDF.
- Manifest assinado (hash SHA-256 + assinatura) e verificado pelo SDK; CDN só com HTTPS.
- Multi-tenant: isolamento por `tenantId` em todas as queries da Theme API; chaves de app por tenant, revogáveis.
- Audit log de quem publicou o quê e quando.

**Acessibilidade (bloqueia a publicação se falhar)**

- Contraste WCAG 2.2 AA em todos os pares semânticos texto/fundo, nos modos claro e escuro.
- Tamanho mínimo de alvo de toque (44 pt iOS, 48 dp Android) não é configurável.
- Tipografia respeita Dynamic Type (iOS) e escala de fonte (Android): tokens definem a base, o sistema escala.
- Foco visível na Web sempre presente, mesmo que o cliente mude cores.

**Governança**

- Um time dono do contrato de tokens (design system), com RFC para tokens novos e changelog público.
- Versões de SDK seguem SemVer; manter suporte a pelo menos 2 versões major de `schemaVersion`.
- Testes de regressão visual (Chromatic na Web, Paparazzi no Android, swift-snapshot-testing no iOS) rodando com temas "extremos" (cores muito claras, raio máximo, fonte grande).

## 11. Roadmap, stack e decisões

Comece pelo contrato e pela Web, prove o modelo num app nativo e só então escale para todas as plataformas e layouts.

**Fases**

1. **Fundação:** JSON Schema v1 dos tokens (≈ 40–60 tokens semânticos), pipeline Style Dictionary, Theme API, CDN + manifest. Critério de saída: um tema publicado gera artefatos para as 5 plataformas.
2. **Studio MVP + Web:** fluxo guiado, editor de cores/tipografia/forma, preview web, validação de contraste, publicar e rollback. Checkout web consumindo `theme.css`. Critério: cliente piloto troca a marca do checkout sem ajuda do time.
3. **SDK nativo piloto:** escolher a plataforma do cliente piloto (RN, iOS ou Android), com cache, fallback e componentes base (botão, input, card). Critério: tema muda no app sem release.
4. **Demais SDKs + preview nativo:** completar iOS, Android, RN, Flutter; app de preview via QR code no Studio.
5. **Layouts:** variantes do checkout (seção 9), depois login e home.
6. **Escala:** aprovação em fluxo, agendamento, A/B de temas, analytics de uso dos tokens.

**Stack recomendada**

| Camada | Tecnologia |
| --- | --- |
| Studio | React, TypeScript, Vite/Next.js, Zustand, Zod, culori, React Native Web |
| Theme API | Node.js (NestJS) ou Go, PostgreSQL (JSONB) |
| Pipeline | Style Dictionary v4 em worker com fila (SQS/BullMQ) |
| Distribuição | S3/GCS + CloudFront/Cloudflare |
| SDK Web | TypeScript, CSS variables, pacote npm |
| SDK React Native | TypeScript, Context + MMKV |
| SDK iOS | Swift Package, SwiftUI + UIKit, Codable |
| SDK Android | Kotlin, Compose + Views, DataStore, Maven |
| SDK Flutter | Dart, `ThemeExtension`, pub.dev |
| Qualidade | Chromatic, Paparazzi, swift-snapshot-testing, testes de contrato do schema |

**ADRs a registrar (decisões já recomendadas aqui)**

- ADR-001: Tokens no padrão W3C DTCG como contrato único entre Studio e apps.
- ADR-002: Três camadas de tokens; componentes consomem apenas semânticos e de componente.
- ADR-003: Entrega híbrida (embarcado + remoto via CDN e manifest).
- ADR-004: Style Dictionary como pipeline de transformação.
- ADR-005: Layouts por variantes pré-aprovadas, sem montagem livre de telas.
- ADR-006: Acessibilidade WCAG AA como bloqueio de publicação.

**Perguntas em aberto**

- Quais plataformas os clientes atuais usam (proporção RN vs nativo vs Flutter)? Isso define a ordem dos SDKs.
- O checkout dentro do app será nativo ou WebView?
- Haverá geração de apps white label inteiros (ícone, nome, publicação na loja) ou só tema dentro de apps existentes?
