---
layout: post
title: "Interview Prep with Vector Embeddings and Cosine Similarity"
date: 2025-11-10 07:07:07 +0100
---

*For all code discussed here, visit the Github repository*  
https://github.com/Sput/interview_assistance

The purpose of this app is to demonstrate the use of vectors and cosine similarity to compare pieces of text. When a new interview question is created, an edge function will create a model answer with the assistance of gpt-4o-mini. After that answer is created, a separate edge function will take that model answer, and turn it into a vector using the gpt-4-embedding model. When a user answers a question, their answer will be turned into a vector by a separate edge function. Another edge function will then perform cosine similarity on the model answer and the users answer to see how similar they are. This is presented as the user's score.

# TL;DR
This project is an interview-practice app that mimics real conversations, letting users read questions, type answers, and get instant AI-based feedback. It combines Next.js, Supabase, python, and embed responses into vectors, and score them by cosine similarity against a model answer. The result is a realistic, feedback-rich system that helps users improve through immediate grading, retries, and conversational learning loops.

# **Practice Interviews, Powered by Voice**

Most interview-prep tools either feel like digital flashcards.

This app takes a different path — it aims to **simulate a real interview**. You read a question, type your answer, get instant feedback and a score — then you try again.

---

## **What You Can Do**

- **Smart question selection:** Get random interview questions without immediate repeats.
- **Automatic scoring:** Receive a 0-100 score based on how semantically close your answer is to an ideal response.
- **Feedback loop:** If your score falls below the bar, the system explains what’s missing — then re-asks the question.
- **At-a-glance score:** A color-coded card shows your latest score (green for pass, red for retry).

---

## **How It Works**

### Create Question

Create questions you want to be prepared for during an interview

1. Model answer will be created for you automatically using Supabase edge functions which calls gpt-4o-mini.  
   ![Model Answer Creation](/images/Screenshot%202025-11-13%20at%207.39.10%20AM.png)

2. After the model answer is created another edge function will create a vector representation of that answer  
   ![Vector Creation](/images/Screenshot%202025-11-13%20at%207.41.49%20AM.png)


### **Conversation Loop**

1. **Ask:** Click “Ask Interview Question” to fetch a category-specific question
2. **Answer:** Type your response
3. **Score & feedback:** Your answer is saved, and converted to a vector representation (same as the model answer above)  
   ![Answer Vectorization](/images/Screenshot%202025-11-13%20at%207.44.05%20AM.png)
4. **Similarity score:** The user answer is then compared to the model answer using cosine similarity.  
   ![Similarity Score](/images/Screenshot%202025-11-13%20at%207.49.36%20AM%201.png)  
   ![Cosine Similarity](/images/Screenshot%202025-11-12%20at%2012.02.40%20PM.png)

### **Scoring Pipeline**

Under the hood, the grading process runs through three stages:

1. **Vectorization:** Both the interview question and its ideal “model” answer are embedded into 1536-dimensional vectors using text-embedding-3-small embedding model
2. **Cosine similarity:** A backend job computes the similarity between your answer’s vector and the reference vector
3. **Score update:** The app displays your score instantly on the **Current Score** card.

### **Random Selection, No Repeats**

To keep practice varied, the app filters out your most recent questions before selecting the next one. If the filtered pool is empty, it falls back to a broader question set.

---

## **Architecture Overview**

### **Frontend**

Built with **ShadCN, Next.js, and React**, the frontend orchestrates the entire answer and feedback flow:

- Conversation view
- Input state management
- Grade renderer

### **Backend**

Powered by **Supabase**, the backend handles persistence and embeddings:

- **Database tables** store questions, answers, and grades.
- **Edge functions:**
  - `make_vectors` — computes embeddings for model answers
  - `answer_vectors` — computes embeddings for user answers
- **Grading job:** calculates cosine similarity between stored vectors

---

## **What’s Next**

- Progress tracking
- Difficulty scaling
- Rubric-based grading
- Adaptive hints
- UI improvements

---

## **Under the Hood: Embeddings & Scoring**

### **Embeddings 101**

Embeddings are numerical vectors representing semantic meaning.

### **Vector Creation**

- Use one embedding model for all text
- Normalize lightly, no heavy cleaning
- For long text, chunk & average vectors
- Store vectors for reuse

### **Cosine Similarity**

![Cosine Similarity](/images/Screenshot%202025-11-12%20at%2012.02.40%20PM.png)

### **Score Mapping**

- Cosine Similarity returns a number between 0-1, which is multiplied by 100

### Summary

This app helps users practice interview questions by generating model answers, embedding them into vectors, and comparing user responses with cosine similarity for instant scoring. It uses a combination of Next.js, Supabase, Python edge functions, and OpenAI embeddings to automate question creation, vectorization, and grading. The result is a realistic, interactive interview simulator that provides immediate feedback, avoids repeat questions, and lays the groundwork for future features like progress tracking and adaptive hints.
