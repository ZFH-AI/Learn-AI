# RAG
## 1、RAG流程
***
分片 -> 索引 -> 召回 -> 重排 -> 生成
***


两个部分：
1. 准备部分(提问前）
    1.1 分片 + 1.2 索引
2. 回答部分(提问后)
    1.3 召回 + 1.4 重排 + 1.5 生成

### 1.1、分片
不同的文本切分策略
1. 固定大小块
   按照预设的字符数或Token数直接切分，比如每500个Token一块，完全不管句子或段落边界在哪
```Python
 from langchain.text_splitter import RecursiveCharacterTextSplitter  

splitter = RecursiveCharacterTextSplitter(  
    chunk_size=500,  
    chunk_overlap=50  
)  

 chunks = splitter.split_text(long_text)
```

   【优点】
        1、速度快、行为可预测，对大规模、结构混乱的数据集很实用
   【缺点】
        1、语义被切散，Embedded的语义表达能力下降
   
    在实践中一般会在相邻分块之间设置一定的重叠来缓解。切分文本时，连续的分块之间通常会加入一小段重叠区域来维持上下文的连贯。所谓重叠，就是前一个分块的尾部几句话，在下一个分块的开头再出现一次，这么做是为了防止跨越分块边界的关键信息丢失。没有重叠的话，检索器可能只拿到部分内容LLM 因此漏掉了关键上下文，给出残缺甚至误导性的回答。重叠量一般控制在分块长度的 10% 到 20%，在冗余和效率之间找一个平衡点。固定大小分块适合的场景包括日志文件、邮件、代码仓库，以及结构参差不齐的大型语料库
<img width="640" height="360" alt="86974ab082064bb6ad5a5b02afe3fe83" src="https://github.com/user-attachments/assets/f1f0fdae-2326-451c-905a-dd713b0375d6" />

2、基于句子的分块
   这种方式按完整句子来划分文本，而不是按任意长度一刀切。每个分块至少包含一个或多个完整的句子，语法完整，语义连贯。好处是每个分块都是一个有意义的思想单元。检索器向 LLM 返回的信息更精确、更易理解，碎片化回答的风险降低不少。实际使用中通常也会搭配小幅重叠，进一步保证分块之间的衔接。

3、基于段落分块
    以完整段落为单位切分，不再拘泥于单个句子或固定 Token 数。这种方式天然保留了文档的结构和行文节奏，检索器更容易抓到完整的想法。每个分块往往对应一个独立的主题或子主题，LLM 处理起来更从容，也更容易给出准确的回答。对长篇文档、研究论文、综述类文章来说，段落级分块效果不错。和句子级分块一样，也可以加重叠来保持连贯

4、语义分割
    语义分块的切入点不是长度，而是语义本身。它利用 Embedding 或相似度分数来识别文本中天然的断裂点——主题切换、上下文转折、章节边界。产出的分块语义清晰度更高，边界和语义对齐，检索质量有明显提升，尤其在知识库、技术文档、结构化文章这类内容上效果突出。代价是计算开销更大而且分块长度不一致，后续处理需要额外考虑。如果文档质量高、主题流转有明确脉络，语义分块往往是精度最高的选择
```Python
 from langchain_experimental.text_splitter import SemanticChunker  
 from sentence_transformers import SentenceTransformer  
   
 model = SentenceTransformer("all-MiniLM-L6-v2")  
 chunker = SemanticChunker(model, breakpoint_threshold=0.4)  
   
 chunks = chunker.split_text(long_text)
```

5、递归分割
    递归分割是固定大小和语义分块之间的一个折中方案。核心思路是优先尊重文档结构，只有在必要时才进一步拆分。具体做法是先尝试按标题切分。如果某个章节还是太长，就按段落切。段落还不够就按句子。句子仍然超限，最后才按字符兜底。这样得到的分块既保有语义完整性，尺寸也在可控范围内。开发者文档、技术手册、学术论文、研究报告——凡是层级结构明确的内容，递归分割都很适合
```Python
 recursive_splitter = RecursiveCharacterTextSplitter(  
     separators=["\n## ", "\n### ", "\n", ". ", ""],  
     chunk_size=600,  
     chunk_overlap=80  
 )  
   
 chunks = recursive_splitter.split_text(long_doc)
```

6、滑动窗口分块
    有些文本的语义天然是跨句分布的。法律合同、科学论文、长段论证，一个完整的意思可能横跨好几个句子。滑动窗口就是为这种场景设计的。它不生成彼此独立的分块，而是创建相互重叠的窗口。比如窗口大小 400 Token，每次滑动 200 Token，这样相邻的分块之间有一半的内容是共享的，语义在边界处不会断裂。上下文保持得很好，但分块数量会膨胀，存储和检索的成本都会上升。法律 RAG、金融分析、医学文献检索、合规审查——这些领域用滑动窗口的比较多。
 
7、层次化分块
    层次化分块是一个多层级的架构：小分块负责细粒度精确检索，中等分块支撑平衡的推理，大分块维持全局上下文。检索时，系统先用小分块锁定精确位置，再把关联的大分块拉进来补充完整上下文。这种组合能有效压制幻觉，提升推理的深度。企业级 RAG 系统和 LlamaIndex 这类多粒度检索框架，背后都有层次化分块的影子。

### 1.2、索引
1. 通过Embedded将片段文本转换为向量
2. 将片段文本和向量文本存入向量数据库中

### 1.3、召回
1. 向量相识度（余弦相识度、欧式距离、点积）

### 1.4、重排
1、cross-encoder

### 1.5、生成

## 实例
```Python
"""
第一步：读取文件，并将文本按行切分
"""
def split_into_chunks(doc_file:str)->list[str]:
    with open(doc_file,"r", encoding="utf-8") as file:
        content = file.read()
    return  [chunk for chunk in content.split("\n")]


chunks = split_into_chunks("逍遥游.txt")
for i , chunk in enumerate(chunks):
    print(f"{i}. {chunk}")


"""
第二步:向量化切分文件
"""
from sentence_transformers import SentenceTransformer

embedding_model = SentenceTransformer(r"D:\PythonProject\text2vec-base-chinese")
def embed_chunk(chunk: str) -> list[float]:
    embedding = embedding_model.encode(chunk, normalize_embeddings=True)
    return embedding.tolist()


embeddings = embed_chunk(chunks)
# print(len(embeddings))
# print(embeddings)


"""
第三步：索引，将向量化存储到向量数据中
"""
import chromadb

chromdb_client = chromadb.EphemeralClient()

chromadb_collection = chromdb_client.get_or_create_collection(name = "rag_collection")

def save_embedding(chunks:list[str], embeddings:list[float]) -> None:
    ids = [str(i) for i in range(len(chunks))]
    chromadb_collection.add(
        documents=chunks,  # 传入字符串列表
        embeddings=embeddings,  # 传入嵌套列表 [ [768...], [768...] ]
        ids=ids  # 传入 ID 列表
    )


save_embedding(chunks, embeddings)

"""
第四步：召回
"""
def retrieve(query: str, top_k: int) -> list[str]:
    query_embedding = embed_chunk(query)
    results = chromadb_collection.query(
        query_embeddings=[query_embedding],
        n_results=top_k
    )
    return results['documents'][0]

query = "哆啦A梦使用的3个秘密道具分别是什么？"
retrieved_chunks = retrieve(query, 5)

for i, chunk in enumerate(retrieved_chunks):
    print(f"[{i}] {chunk}\n")

"""
第五步：重排
"""
from sentence_transformers import CrossEncoder

def rerank(query: str, retrieved_chunks: list[str], top_k: int) -> list[str]:
    cross_encoder = CrossEncoder('cross-encoder/mmarco-mMiniLMv2-L12-H384-v1')
    pairs = [(query, chunk) for chunk in retrieved_chunks]
    scores = cross_encoder.predict(pairs)

    scored_chunks = list(zip(retrieved_chunks, scores))
    scored_chunks.sort(key=lambda x: x[1], reverse=True)

    return [chunk for chunk, _ in scored_chunks][:top_k]

reranked_chunks = rerank(query, retrieved_chunks, 3)

for i, chunk in enumerate(reranked_chunks):
    print(f"[{i}] {chunk}\n")


"""

"""
from dotenv import load_dotenv
from google import genai

load_dotenv()
google_client = genai.Client()

def generate(query: str, chunks: List[str]) -> str:
    prompt = f"""你是一位知识助手，请根据用户的问题和下列片段生成准确的回答。

用户问题: {query}

相关片段:
{"\n\n".join(chunks)}

请基于上述内容作答，不要编造信息。"""

    print(f"{prompt}\n\n---\n")

    response = google_client.models.generate_content(
        model="gemini-2.5-flash",
        contents=prompt
    )

    return response.text

answer = generate(query, reranked_chunks)
print(answer)
```

