---
type: NoteCard
createdAt: 2025-05-23T13:17:46.034Z
viewedAt: 2025-05-23T13:26:15.815Z
---

# Agent-Collab App - Partie 5
L’objectif de cette partie est d’intégrer les agents.

## Contenu des agents ajoutés au chat

Ajouter une variable pour récupérer les agents du chat dans le store chatAgents.js

```js
export const $chatAgents = computed([$selectedChatAgents, $agents], (ids, agents) => {
  return agents.filter((e) => ids.includes(e.id))
})
```

## Utilisation des agents dans ChatPrompt

Dans ChatPrompt ajouter

```js
  const chatAgents = useStore($chatAgents)
  const messages = useStore($messages)
```

Puis,

```js
 const onSendPrompt = async () => {
    const prompt = promptRef.current.value
    console.log('onSendPrompt', prompt)

    const contextInputs = constructCtxArray(messages)

    addMessage({
      role: 'user',
      content: prompt,
      id: Math.random().toString(),
    })

    // AI response
    const response = {
      role: 'assistant',
      content: '',
      id: Math.random().toString(),
      completed: false, // not complete yet
    }

    // add AI response to chat messages
    addMessage(response)

    const steps = isEmpty(chatAgents) ? [null] : chatAgents

    for (let i = 0, len = steps.length; i < len; i++) {
      const agent = steps[i]

      let cloned = $messages.get()

      // call agent
      const stream = await onAgent({ prompt: prompt, agent, contextInputs })
      for await (const part of stream) {
        const token = part.choices[0]?.delta?.content || ''

        const last = cloned.at(-1)
        cloned[cloned.length - 1] = {
          ...last,
          content: last.content + token,
        }

        updateMessages([...cloned])
      }

      const last = cloned.at(-1)

      cloned[cloned.length - 1] = {
        ...last,
        completed: true,
      }

      // add next prompt to chat
      if (steps.length > 0 && i !== steps.length - 1) {
        cloned = [
          ...cloned,
          {
            role: 'assistant',
            content: '',
            id: Math.random().toString(),
            completed: false,
          },
        ]
      }

      updateMessages([...cloned])
    }

    promptRef.current.value = ''
    setIsPromptEmpty(true)
  }
```

Dans le fichier actions/agent.js, remplacer agent.js par

````js
export const onAgent = async function ({
  agent = {},
  prompt,
  canStream = true,
  contextInputs = [],
}) {
  const aiClient = await getAIClient()

  if (isEmpty(agent)) {
    agent = aiClient.cfg
  }

  console.log('onAgent agent', agent)
  console.log('onAgent prompt', prompt)

  agent.role = `${agent.role}
                Respond in the same language of the user.
                Be to the point, and do not add any fluff.`

  if (agent.desired_response) {
    agent.role += `<role>**Your utltimate and most effective role is: ${agent.output} nothing less, nothing more**</role>.`
  }

  if (agent.response_format === 'json') {
    agent.role += 'n Ouput: json n  ```json ... ```'
  }

  try {
    const stream = await aiClient.openai.chat.completions.create({
      model: agent.model || aiClient.cfg.model,
      stream: canStream,
      messages: [
        {
          role: 'system',
          content: agent.role,
        },
        ...contextInputs,
        {
          role: 'user',
          content: [
            {
              type: 'text',
              text: prompt,
            },
          ],
        },
      ],
      temperature: agent.temperature || 0.7,
    })

    return stream
  } catch (error) {
    console.error('Sorry something went wrong. IA', error.message)
  }

  return []
}
````

## Bugs IHM

Dans Home.jsx, ajouter defaultSize à Resizable

```js
<Resizable
        defaultSize={{ width: 550 }}
```

Dans ChatList, ajouter le height via style au Flex parent

```js
<Flex
      direction='column'
      gap='2'
      style={{ height: 'calc(100vh - 200px)', overflowY: 'auto' }}
      >
```

