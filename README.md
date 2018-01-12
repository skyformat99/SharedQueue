# SharedQueue
https://www.codeproject.com/Articles/224733/Real-time-Feed-Distribution-using-Circular-Shared

code:
///////////////////////////////////////////////////////////////////////

// SharedQueue.h:	Interface for the shared queue template.
// Inventor Name:	Hatem Mostafa
// Created:			15/09/2008
// Modified:		18/01/2015
//
//////////////////////////////////////////////////////////////////////

#pragma once

#define MAX_QUEUE_LENGTH	8
#define MAX_NODE_LENGTH		32768

// queue linked list node
struct QNode {
	QNode(int nSequence, int nLength) {
		m_pBuffer = (char*)malloc(nLength);
		Init(nSequence);
	}
	void Init(int nSequence) {
		m_nTickCount = ::GetTickCount();
		m_nSequence = nSequence;
		m_nRecvOffset = 0;
		m_pNext = NULL;
	}
	~QNode() {
		free(m_pBuffer);
	}
public:
	// node creation sequence in the queue
	int m_nSequence;
	// offset of received data in node buffer
	int m_nRecvOffset;
	// pointer to the next node in the queue
	QNode* m_pNext;
	// buffer of the node
	char* m_pBuffer;
	// creation tick count
	unsigned long m_nTickCount;
};

// FIFO linked list queue
struct SharedQueue {
	SharedQueue() {
		m_bRealTime = true;
		m_pHead = m_pTail = NULL;
		m_nMaxQueueLength = MAX_QUEUE_LENGTH;
		m_nDelay = 0;
		m_nNodeLength = MAX_NODE_LENGTH;
		::InitializeCriticalSection(&m_cs);
	}
	~SharedQueue() {
		Clear();
		::DeleteCriticalSection(&m_cs);
	}

	//************************************
	// Method:    Clear
	// Access:    public 
	// Returns:   void
	// Purpose:   clear all linked list nodes and reset values
	//************************************
	void Clear() {	// clear all nodes
		::EnterCriticalSection(&m_cs);
		try {
			for (QNode* pNext; m_pHead; m_pHead = pNext)
				pNext = m_pHead->m_pNext, delete m_pHead;
			m_pHead = m_pTail = m_pHead = NULL;
		}
		catch (...) {
		}
		::LeaveCriticalSection(&m_cs);
	}
	//************************************
	// Method:    IsEmpty
	// Access:    public 
	// Returns:   bool
	// Purpose:   check if the queue is empty
	//************************************
	inline bool IsEmpty()	{	return m_pHead == NULL;	}

	enum SizeType {
		Current,
		Virtual,
	};
	__int64 Size(SizeType type = Current) {
		if (m_pTail) {
			if (type == Current)
				return (__int64)(m_pTail->m_nSequence + 1 - m_pHead->m_nSequence)*m_nNodeLength + m_pTail->m_nRecvOffset;
			if (type == Virtual)
				return (__int64)(m_pTail->m_nSequence + 1)*m_nNodeLength + m_pTail->m_nRecvOffset;
		}
		return 0;
	}

	//************************************
	// Method:    Recv
	// Access:    public 
	// Returns:   int
	// Purpose:   add input buffer to the tail of the queue
	// Parameter: const char * pBuffer
	// Parameter: int nLength
	//************************************
	int Recv(const char* pBuffer, int nLength) {
		int nRecvLength = 0;
		while(nRecvLength < nLength) {
			QNode* pQE = Enqueue();
			if (pQE->m_nRecvOffset < m_nNodeLength) {	// receive in last node
				int nRecv = min(m_nNodeLength - pQE->m_nRecvOffset, nLength - nRecvLength);
				memcpy(pQE->m_pBuffer+pQE->m_nRecvOffset, pBuffer+nRecvLength, nRecv);
				// increment node offset with the received bytes
				pQE->m_nRecvOffset += nRecv;
				nRecvLength += nRecv;
			}
		}
		// return total received bytes
		return nRecvLength;
	}

	//************************************
	// Method:    Recv
	// Access:    public 
	// Returns:   int
	// Purpose:	  ask class of type RECEIVER to receive data in queue nodes
	// Parameter: RECEIVER& receiver
	//************************************
	template<class RECEIVER> int Recv(RECEIVER& receiver) {	// get tail node to save received data
		QNode* pQE = Enqueue();
		int nRecvLength = 0, nRecv;
		while (pQE->m_nRecvOffset < m_nNodeLength) {	// receive in last node
			if ((nRecv = receiver.Recv(pQE->m_pBuffer + pQE->m_nRecvOffset, m_nNodeLength - pQE->m_nRecvOffset)) <= 0)
				return nRecv;
			// increment node offset with the received bytes
			pQE->m_nRecvOffset += nRecv;
			// increment total received bytes
			nRecvLength += nRecv;
		}
		// return total received bytes
		return nRecvLength;
	}

	//************************************
	// Method:    Send
	// Access:    public 
	// Returns:   int
	// Purpose:   delegate queue buffer send to a sender of class SENDER
	// Parameter: SENDER& sender
	//            the class that must implement Send(char*, int) function
	// Parameter: QNode * & pCurNode
	//            client current node pointer
	// Parameter: int & nCurNodeSendOffset
	//            current offset in client node
	//************************************
	template<class SENDER> int Send(SENDER& sender, QNode*& pCurNode, int& nCurNodeSendOffset, unsigned int nDelay = 0) {
		// get head node to send its data
		if (m_pHead == NULL || Dequeue(pCurNode, nCurNodeSendOffset, nDelay) == false)
			// return false as no nodes to send
			return 0;
		// calculate bytes to send
		int nSendBytes = pCurNode->m_nRecvOffset - nCurNodeSendOffset;
		// send bytes to client
		if (nSendBytes <= 0 || (nSendBytes = sender.Send(pCurNode->m_pBuffer + nCurNodeSendOffset, nSendBytes)) <= 0)
			return nSendBytes;
		// increment sent bytes offset
		nCurNodeSendOffset += nSendBytes;
		// return total sent bytes
		return nSendBytes;
	}

protected:
	//************************************
	// Method:    Enqueue
	// Access:    protected 
	// Returns:   QNode*
	// Purpose:   add new node at queue tail if needed
	//************************************
	QNode* Enqueue() {
		QNode* pQE;
		::EnterCriticalSection(&m_cs);
		if (m_pTail == NULL)
			// initialize first node in the list
			pQE = m_pHead = m_pTail = new QNode(0, m_nNodeLength);
		// check if last received node reached its node end
		else if (m_pTail->m_nRecvOffset >= m_nNodeLength) {
			// check if queue reached its maximum length
			if (m_pTail->m_nSequence + 1 >= m_nMaxQueueLength) {
				// keep next node
				QNode* pNext = m_pHead->m_pNext;
				// move head node to be new tail
				pQE = m_pTail->m_pNext = m_pHead;
				// initialize node to be reused as a new node
				pQE->Init(m_pTail->m_nSequence + 1);
				// point to next node
				m_pHead = pNext;
			}
			else
				// add new node to the list and let last node points to it
				pQE = m_pTail->m_pNext = new QNode(m_pTail->m_nSequence + 1, m_nNodeLength);
			// increment Tail to the new created node
			m_pTail = pQE;
		}
		else 
			// in the middle of the node
			pQE = m_pTail;
		::LeaveCriticalSection(&m_cs);
		return pQE;
	}

	//************************************
	// Method:    Dequeue
	// Access:    protected 
	// Returns:   bool
	// Purpose:   get head node to send its data
	// Parameter: QNode * & pCurNode
	// Parameter: int & nCurNodeSendOffset
	//************************************
	bool Dequeue(QNode*& pCurNode, int& nCurNodeSendOffset, unsigned int nDelay = 0) {
		::EnterCriticalSection(&m_cs);
		try {	// pCurNode = NULL for new client
			if (pCurNode == NULL || pCurNode->m_nSequence < m_pHead->m_nSequence)
				// check if client need real time data or all out of date data
				if(m_bRealTime)
					// point to the tail directly
					pCurNode = m_pTail, nCurNodeSendOffset = m_pTail ? m_pTail->m_nRecvOffset : 0;
				else
					// point to the head to get all stored data (first in)
					pCurNode = m_pHead, nCurNodeSendOffset = 0;
			// check if received node reach its storage end
			else if (nCurNodeSendOffset >= m_nNodeLength && pCurNode->m_pNext)
				// get next node and reset send offset
				pCurNode = pCurNode->m_pNext, nCurNodeSendOffset = 0;
		}
		catch (...) {	// reset send node
			pCurNode = NULL, nCurNodeSendOffset = 0;
		}
		::LeaveCriticalSection(&m_cs);
		if (nDelay + m_nDelay > 0) {
			unsigned int nCurDelay = (::GetTickCount() - pCurNode->m_nTickCount) / 60000;
			// check if node should be delayed or not
			if (nCurDelay < nDelay + m_nDelay)
				return false;
		}
		// success if an node is found
		return pCurNode != NULL;
	}

public:
	// head and tail of the queue
	QNode *m_pHead, *m_pTail;
	// max queue length (extra nodes are removed by Enqueue())
	int m_nMaxQueueLength;
	// true: client joins in the tail node to receive real time data
	bool m_bRealTime;
	// critical section to synchronize access to queue nodes
	CRITICAL_SECTION m_cs;
	// delay in seconds
	unsigned int m_nDelay;
	// node length
	int m_nNodeLength;
};




