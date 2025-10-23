---
title: Como reduzi 90% do bundle JavaScript com Next.js Server Actions
date: 2025-10-23
excerpt: Reduzi 90% do JavaScript do meu bundle migrando de chamadas client-side para Server Actions no Next.js. Neste artigo, mostro o antes e depois, explico o que são Server Actions e como o useActionState simplifica a lógica de formulários enquanto melhora performance e DX.
tags: [nextjs, react, frontend, optimization, server-actions]
---

Construí algumas partes da minha plataforma Git to Know Me em vibe coding e sempre tenho o costume de refatorar bastante quando faço uso essa prática. Pois bem, enquanto fazia isso, percebi que a page de Dashboard estava pesada: 62kB de JS. Isso não é absurdo em uma aplicação complexa, mas estamos falando de um app simples, que ainda está engatinhando. Então fui investigar.

---

## 1. Identificando o gargalo

O formulário do Dashboard estava usando um padrão clássico de fetch client-side: tudo acontecia no browser — validação, chamada à API e atualização da UI.

```typescript
"use client";

export function DashboardForm({ config }) {
  const [isSaving, setIsSaving] = useState(false);
  
  async function handleSave() {
    setIsSaving(true);
    
    // Validação client-side
    const validation = configSchema.safeParse(data);
    if (!validation.success) {
      toast.error(validation.error.issues[0].message);
      return;
    }
    
    // Fetch manual
    const res = await fetch("/api/config", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(config),
      credentials: "include",
    });
    
    const data = await res.json();
    
    if (res.ok) {
      toast.success("Salvo!");
      router.refresh();
    } else {
      toast.error(data.error);
    }
    
    setIsSaving(false);
  }
  
  return (
    <form onSubmit={handleSave}>
      {/* campos do form */}
      <button disabled={isSaving}>
        {isSaving ? "Salvando..." : "Salvar"}
      </button>
    </form>
  );
}
```

E aí estava a raiz de tudo:

* O **Zod** estava sendo carregado no bundle do client;
* O **fetch manual** adicionava código redundante;
* O **estado local** (loading, erros, etc.) trazia mais complexidade;
* E sem **progressive enhancement**, nada funcionava sem JS.

👉🏻 **Resultado:** 62kB de JavaScript apenas para enviar um formulário.

---

## 2. A solução: mover para o servidor

Sabendo disso, pensei: “isso aqui é perfeito para **Server Actions**”.

### ✔️ O que são Server Actions?

As **Server Actions** são uma das features mais poderosas introduzidas no Next.js.
Elas permitem que você **execute lógica do servidor diretamente a partir do client**, sem precisar criar rotas API ou lidar com `fetch()` manualmente.

Em resumo:

* Você escreve uma função **marcada com `"use server"`**;
* O Next.js cuida de toda a serialização e chamada dessa função a partir do form ou do componente;
* Tudo roda no **servidor**, e o **cliente** só envia os dados via form (sem precisar carregar a lógica toda).

A dupla perfeita: simplicidade e performance!

---

Comecei refatorando o código da ação:

```typescript
// app/actions/config.ts
"use server";

const configSchema = z.object({
  bio: z.string().max(240),
  // outros campos...
});

export async function updateConfigAction(
  prevState: unknown, 
  formData: FormData
) {
  const session = await getServerSession();
  
  if (!session?.user) {
    return { error: "Não autenticado" };
  }
  
  // Validação no servidor ✅
  const result = configSchema.safeParse({
    bio: formData.get("bio"),
    // ...
  });
  
  if (!result.success) {
    return { error: result.error.issues[0].message };
  }
  
  await setUserConfig(session.user.username, result.data);
  revalidatePath("/dashboard");
  
  return { success: "Salvo com sucesso!" };
}
```

E no componente do client, deixei o mínimo possível:

```typescript
// app/dashboard/DashboardForm.tsx
"use client";

import { useActionState } from "react";
import { updateConfigAction } from "../actions/config";

export function DashboardForm({ config }) {
  const [state, formAction] = useActionState(updateConfigAction, null);
  
  useEffect(() => {
    if (state?.success) toast.success(state.success);
    if (state?.error) toast.error(state.error);
  }, [state]);
  
  return (
    <form action={formAction}>
      {/* campos do form */}
      <button type="submit">Salvar</button>
    </form>
  );
}
```

### ✔️ Entendendo o useActionState

O hook **`useActionState`** é uma adição recente ao React (e muito usada com Server Actions).
Ele facilita a gestão de formulários sem precisar de `useState`, `useEffect` ou bibliotecas externas.

Basicamente, ele:

* Recebe a **ação do servidor** (`updateConfigAction`) e um estado inicial;
* Retorna o **estado atual** (resultado da action) e uma **função de envio** (`formAction`);
* Atualiza o estado automaticamente quando a ação retorna algo novo.

Ou seja, ele faz o mesmo papel de um `useState`, mas sincronizado com a resposta da **Server Action**.
Perfeito para formulários sem muita complexidade.

---

## 3. Resultado

Assim que refatorei e rodei novamente a aplicação, o bundle despencou:
**de 62kB para apenas 5.7kB.**

👉🏻 Isso significa uma **redução de 90,8% no JavaScript do client**, além de:

* Validação segura no servidor (sem expor schema ou lógica);
* Menos dependências carregadas;
* Aplicação mais rápida;
* E suporte a **progressive enhancement** — o form funciona até com JS desabilitado.

---

Estou trazendo essa refatoração justamente para te mostrar como as Server Actions mudam completamente a forma de pensar aplicações com Next.js.
Elas simplificam o fluxo, reduzem o código enviado para o client e deixam o servidor assumir o que ele faz de melhor: processar, validar e responder.

O Next.js já entrega ferramentas incríveis para otimização e produtividade. O segredo está em conhecê-las e usá-las de forma inteligente, aproveitando ao máximo o que o framework oferece. **Nem toda funcionalidade React deve ser escrita da mesma forma!** Lembre-se disso.

