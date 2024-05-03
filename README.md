# Retrieval-Augmented Generation (RAG)

تولید تقویت شده با بازیابی

Hot Topic among developers

## **Unlocking AI's Potential with Less**

- Train AI with no resources and limited knowledge
- How does it work ?

## The how ?

- Imagine a librarian (the retrieval system) teamed up with a storyteller (the generative model).
- Provide specific data for language model

## Examples

- **Customer Support Chatbots**
- **Legal Research Tools**
- **Health Advisory Systems**
- Education
- **Content Recommendation Engines**
- **Crisis Management Tools**

## Let’s make one:

### DanGPT

### requirements

1. Data (Vectorized)
2. Database
3. A Language model API

## What is Vectorized Data

Vectorizing is like turning words into numbers. Imagine each word as a dot in space. Words that have similar meanings are placed closer together. This helps computers quickly understand and find the right words to use when answering questions or making suggestions.

- **Efficient Retrieval**:  quickly retrieve information by computing similarities between vectors.
- **Semantic Understanding**: Vectors help capture the semantic meaning of words, phrases, or entire documents. This means the system can understand context and retrieve information that is not just a direct match but contextually relevant.

## **How to Vectorize Data**

### Methods of Data Vectorization

1. **Word Embeddings**:
    - **Tools**: Models like Word2Vec or GloVe
2. **Sentence and Document Embeddings**:
    - **Tools**: BERT or Doc2Vec.
    - **How it Works**: They look at how words relate to each other in a sentence to understand the bigger picture
3. **Using OpenAI's text-embedding-3-small**:
    - **Description**: This is a specific tool developed by OpenAI that's really good at turning larger chunks of text—like paragraphs—into vectors.

```jsx
import OpenAI from "openai";

export const text2vec = async (texts: string[]) => {
  const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

  const embedding = await openai.embeddings.create({
    model: "text-embedding-3-small",
    dimensions: 1024,
    input: texts,
  });

  return embedding.data.map((d) => d.embedding);
};

```

We then store those vectors in a vector database. Our weapon of choice here is **AstraDB (**A free database provider with vectorization support**)**

![Untitled.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/c0fb2397-b7b8-4def-8c46-89e99ed039b2/Untitled.jpg)

![24.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/da2a8d2a-e91e-4b78-b390-46c5f56922d1/24.jpg)

![24.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/e3a800f1-719e-4165-813d-eb57f835042b/24.jpg)

## Similarity metrics in AstraDB

- **Cosine**
    
    Cosine does not require vectors to be normalized.
    
    ![sim-astra-cosine2 (1).png](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/d1097eb3-fb7e-41e9-ae61-6265480c0b89/sim-astra-cosine2_(1).png)
    
    - 0: vectors are diametrically opposed
    - 0.5: have no match
    - 1: identical in direction
- **Dot product**
    
    The dot product algorithm is ~50% faster than cosine, but requires vectors to be normalized.
    
    ![given-two-vectors.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/4ac44a04-59b3-4fb0-8200-adef0cd64551/given-two-vectors.png)
    
    ![dot-product-calculation.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/87293874-c043-44bb-a827-bf75fd5dbb0d/dot-product-calculation.png)
    
    represents the cosine of the angle between the two vectors.
    
    In high-dimensional vector spaces, such as those produced by embedding algorithms or neural networks
    
- **Euclidean metric**
    
    ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/b9002cf6-a8d8-481f-b832-338b45751b0d/Untitled.png)
    
    ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/f95f3531-17c4-475c-a771-b7088eef117e/Untitled.png)
    
    ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/d06d0829-429b-4daa-b437-a53e6f8c8509/Untitled.png)
    

### Then, when a user submits a query, we:

1. Turn the query into a vector using the same embeddings model.
2. Search the vector database for the most similar vectors to the query vector, or vectors "near" the query vector in dimensional space.
3. Retrieve many original texts from the most similar vectors.

```jsx
const results = (
    await collection
      .find(
        {},
        { sort: { $vector: $vector! }, limit: 10, includeSimilarity: true }
      )
      .toArray()
  ).filter((r) => r.$similarity > 0.76);
```

1. Take those original texts and feed them as context into a generative AI model. In this case, OpenAI's `gpt-3.5-turbo`. 

```jsx
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

 const response: any = await openai.chat.completions.create({
      model: "gpt-3.5-turbo",
      stream: Boolean(stream),
      messages: await getMessages(
        results.map((r) => r.text),
        query
      ),
      openpipe: {
        logRequest: true,
      },
    });

const getMessages = async (results: string[], query: string) => {
  let resultsCopy = [...results];
  const getSystemPrompt = (
    r: string[]
  ) => `You are Dan Abramov, a former engineer on the React.js core team at Meta. You are the leading global expert on React.js and have been asked to answer a question. Use this context to best answer the question. Respond in THE SAME TEXT STYLE as the context below. Do not use any of your own knowledge about these topics but instead, ONLY use the context you are given. Answer in about 1 paragraph, no more. Here is your context:
${r.length >= 3 ? r.join("\n- ") : "Answer EXACTLY LIKE THIS 'i dont know'"}`;
```

1. The generative AI model then generates a response based on the context it was given, prentending to be Dan.

## How to buy gpt api

**Buy Account + Charges: Iranicart**

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/e68bcd12-3bb0-432a-a8fa-8c18d79ef800/Untitled.png)

**Make an API key in OpenAI**

![xx.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/8fb67298-3f03-4db9-bbb0-f95054ae068e/xx.jpg)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/64b00c65-e76d-477d-9469-a4c21b3532ab/feb99b30-3b4f-4e0a-a3e0-00707e26717a/Untitled.png)

### Test [danGPT here](https://dangpt.vercel.app/)