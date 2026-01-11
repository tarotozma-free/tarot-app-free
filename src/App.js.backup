import React, { useState, useEffect, useRef } from 'react';
import { createClient } from '@supabase/supabase-js';
import { ArrowLeft } from 'lucide-react';
import './App.css';

// ========== 버전 설정 ==========
const APP_VERSION = 'FREE';

// Supabase 설정 - 환경변수에서 가져오기
const SUPABASE_URL = process.env.REACT_APP_SUPABASE_URL;
const SUPABASE_ANON_KEY = process.env.REACT_APP_SUPABASE_ANON_KEY;
const GEMINI_API_KEY = process.env.REACT_APP_GEMINI_API_KEY;

// 환경변수 체크
if (!SUPABASE_URL || !SUPABASE_ANON_KEY || !GEMINI_API_KEY) {
  console.error('환경변수가 설정되지 않았습니다!');
  console.log('VITE_SUPABASE_URL:', SUPABASE_URL ? '✅' : '❌');
  console.log('VITE_SUPABASE_ANON_KEY:', SUPABASE_ANON_KEY ? '✅' : '❌');
  console.log('VITE_GEMINI_API_KEY:', GEMINI_API_KEY ? '✅' : '❌');
}

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

// ========== 사용자 ID 관리 ==========
const getUserId = () => {
  let userId = localStorage.getItem('tarot_user_id');
  
  if (!userId) {
    userId = 'free_' + Date.now() + '_' + Math.random().toString(36).substr(2, 9);
    localStorage.setItem('tarot_user_id', userId);
    console.log('새로운 사용자 ID 생성:', userId);
  }
  
  return userId;
};

function App() {
  const [step, setStep] = useState('loading');
  const [userId, setUserId] = useState('');
  const [userName, setUserName] = useState('');
  const [tempName, setTempName] = useState('');
  const [concern, setConcern] = useState('');
  const [sessionTitle, setSessionTitle] = useState('');
  const [displayTitle, setDisplayTitle] = useState('');
  const [messages, setMessages] = useState([]);
  const [isTyping, setIsTyping] = useState(false);
  const [drawnCards, setDrawnCards] = useState([]);
  const [streamingMessage, setStreamingMessage] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);
  const [allCards, setAllCards] = useState([]);
  const [currentSessionId, setCurrentSessionId] = useState(null);
  const [pastSessions, setPastSessions] = useState([]);
  const [visitCount, setVisitCount] = useState(0);
  const [currentCardIndex, setCurrentCardIndex] = useState(0); // 현재 뽑고 있는 카드 인덱스
  const [finalReadingComplete, setFinalReadingComplete] = useState(false); // 총평 완료 여부
  const messagesEndRef = useRef(null);

  useEffect(() => {
    initializeUser();
  }, []);

  useEffect(() => {
    scrollToBottom();
  }, [messages, streamingMessage]);

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  const initializeUser = async () => {
    try {
      const id = getUserId();
      setUserId(id);

      const count = parseInt(localStorage.getItem('tarot_visit_count') || '0') + 1;
      localStorage.setItem('tarot_visit_count', count.toString());
      setVisitCount(count);

      const savedName = localStorage.getItem('tarot_user_name');
      
      if (savedName) {
        setUserName(savedName);
        await loadUserData(id);
        setStep('input');
      } else {
        setStep('name_input');
      }

      console.log('카드 로딩 시작...');
      // 만신카드만 필터링
      const { data: cards, error } = await supabase
        .from('tarot_cards')
        .select('*')
        .eq('card_type', '만신카드')
        .order('card_num');
      
      if (error) {
        console.error('카드 로드 오류 상세:', error);
        alert('카드 데이터를 불러오는데 실패했습니다.\n\n에러: ' + error.message);
      } else if (cards && cards.length > 0) {
        console.log('만신카드 로드 성공:', cards.length + '장');
        setAllCards(cards);
      } else {
        console.error('만신카드 데이터가 비어있습니다');
        alert('만신카드 데이터가 없습니다. DB를 확인해주세요.');
      }
    } catch (err) {
      console.error('초기화 오류:', err);
      alert('초기화 중 오류가 발생했습니다: ' + err.message);
      setStep('name_input');
    }
  };

  const loadUserData = async (userId) => {
    try {
      const { data: sessions, error } = await supabase
        .from('consultations')
        .select('*')
        .eq('free_user_id', userId)
        .eq('version_type', 'free')
        .order('created_at', { ascending: false })
        .limit(10);
      
      if (error) {
        console.error('과거 상담 로드 오류:', error);
      } else if (sessions) {
        setPastSessions(sessions);
        console.log('과거 상담 로드:', sessions.length + '개');
      }
    } catch (err) {
      console.error('데이터 로드 오류:', err);
    }
  };

  const handleNameSubmit = async () => {
    if (!tempName.trim()) {
      alert('이름을 입력해주세요!');
      return;
    }

    localStorage.setItem('tarot_user_name', tempName);
    setUserName(tempName);

    try {
      const { error } = await supabase
        .from('free_users')
        .insert([{
          free_user_id: userId,
          name: tempName,
          visit_count: 1
        }]);
      
      if (error) {
        console.error('사용자 저장 오류:', error);
      } else {
        console.log('사용자 정보 저장 완료');
      }
    } catch (err) {
      console.error('사용자 저장 오류:', err);
    }

    setStep('input');
  };

  const callGeminiAPI = async (prompt) => {
    try {
      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key=${GEMINI_API_KEY}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }]
          })
        }
      );
      
      const data = await response.json();
      
      if (data.candidates && data.candidates[0]) {
        return data.candidates[0].content.parts[0].text;
      }
      
      return '응답을 받지 못했습니다.';
    } catch (error) {
      console.error('Gemini 오류:', error);
      return '연결 오류가 발생했습니다.';
    }
  };

  const getOpeningMessage = async (question) => {
    const prompt = `다음 타로 질문을 읽고, 타로 마스터가 카드 섞기 전에 할 자연스러운 멘트를 만드세요.

질문: "${question}"

요구사항:
- 질문의 핵심 주제 파악
- "~에 대한 고민이시군요. 카드를 섞어보겠습니다" 형식
- 30자 이내
- 따뜻하고 공감하는 톤

멘트:`;
    
    try {
      const response = await callGeminiAPI(prompt);
      return response.trim().replace(/["']/g, '');
    } catch (err) {
      return "고민이 느껴지네요. 카드를 섞어보겠습니다.";
    }
  };

  const handleStartConsultation = async () => {
    if (!concern.trim()) {
      alert('고민을 입력해주세요!');
      return;
    }

    if (allCards.length === 0) {
      alert('카드 데이터를 불러오는 중입니다. 잠시 후 다시 시도해주세요.');
      return;
    }

    setStep('consultation');
    setCurrentCardIndex(0);
    
    try {
      const displayPrompt = `다음 질문을 100자 이내로 자연스럽게 요약:
"${concern}"
핵심만 간결하게:`;
      const displaySummary = await callGeminiAPI(displayPrompt);
      setDisplayTitle(displaySummary.trim().replace(/["']/g, '').substring(0, 100));
    } catch (err) {
      setDisplayTitle(concern.substring(0, 100));
    }

    const openingMsg = await getOpeningMessage(concern);
    setMessages([
      { role: 'assistant', content: openingMsg }
    ]);

    setSessionTitle(concern.substring(0, 30) + (concern.length > 30 ? '...' : ''));

    setTimeout(() => {
      addMessage('assistant', `${userName}님의 타로 상담을 시작합니다.`);
    }, 1000);

    setTimeout(() => {
      drawAllCardsAtOnce();
    }, 2000);
  };

  // 3장 한번에 뽑기 (로딩 1회)
  const drawAllCardsAtOnce = async () => {
    addMessage('assistant', '카드를 섞고 있습니다...');
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    const shuffled = [...allCards].sort(() => Math.random() - 0.5);
    const selectedCards = shuffled.slice(0, 3);
    
    setDrawnCards(selectedCards);
    console.log('3장 뽑기 완료:', selectedCards);
    
    // 첫 번째 카드부터 하나씩 보여주기
    setTimeout(() => {
      revealAndInterpretCard(0, selectedCards);
    }, 1000);
  };

  // 카드 하나씩 공개하고 해석
  const revealAndInterpretCard = async (cardIndex, allSelectedCards) => {
    const card = allSelectedCards[cardIndex];
    const cardLabel = cardIndex === 0 ? '첫 번째' : cardIndex === 1 ? '두 번째' : '세 번째';
    
    addMessage('assistant', `${cardLabel} 카드: ${card.name}`);
    
    setTimeout(() => {
      interpretCard(cardIndex, allSelectedCards);
    }, 1000);
  };

  // 개별 카드 해석 (간결하고 자연스럽게)
  const interpretCard = async (cardIndex, allSelectedCards) => {
    setIsTyping(true);
    setIsStreaming(true);

    const card = allSelectedCards[cardIndex];
    let prompt;
    
    if (cardIndex === 0) {
      // 과거/현재 위치
      prompt = `당신은 친근하고 따뜻한 타로 상담가입니다.

내담자 고민: "${concern}"

뽑힌 카드: ${card.name}
키워드: ${card.keyword}
의미: ${card.meaning}

이 카드는 **과거/현재 상황**을 나타냅니다.
${card.name} 카드가 보여주는 현재 상황을 자연스럽게 설명해주세요.

요구사항:
- 친구에게 말하듯 자연스럽고 따뜻하게
- 100자 내외로 간결하게
- AI투 딱딱한 말투 금지
- "~입니다", "~됩니다" 같은 격식체보다는 "~네요", "~같아요" 사용`;
      
    } else if (cardIndex === 1) {
      // 내면/감정 위치
      prompt = `당신은 친근하고 따뜻한 타로 상담가입니다.

내담자 고민: "${concern}"

뽑힌 카드: ${card.name}
키워드: ${card.keyword}
의미: ${card.meaning}

이 카드는 **내면의 감정/잠재의식**을 나타냅니다.
${card.name} 카드가 보여주는 내담자의 숨겨진 마음을 설명해주세요.

요구사항:
- 친구에게 말하듯 자연스럽고 따뜻하게
- 100자 내외로 간결하게
- 앞 카드 내용 반복 금지
- "~네요", "~같아요" 사용`;
      
    } else {
      // 미래/결과 위치
      prompt = `당신은 친근하고 따뜻한 타로 상담가입니다.

내담자 고민: "${concern}"

뽑힌 카드: ${card.name}
키워드: ${card.keyword}
의미: ${card.meaning}

이 카드는 **미래/결과**를 나타냅니다.
${card.name} 카드가 보여주는 앞으로의 흐름을 설명해주세요.

요구사항:
- 친구에게 말하듯 자연스럽고 따뜻하게
- 100자 내외로 간결하게
- 앞 카드 내용 반복 금지
- "~네요", "~같아요" 사용`;
    }

    try {
      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key=${GEMINI_API_KEY}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }],
            generationConfig: {
              temperature: 0.9,
              topP: 0.95,
              topK: 40,
              maxOutputTokens: 200
            }
          })
        }
      );

      const data = await response.json();
      const fullText = data.candidates[0].content.parts[0].text;

      // 스트리밍 효과
      let currentText = '';
      for (let i = 0; i < fullText.length; i++) {
        currentText += fullText[i];
        setStreamingMessage(currentText);
        await new Promise(resolve => setTimeout(resolve, 20));
      }

      addMessage('assistant', fullText);
      setStreamingMessage('');
      
      setIsStreaming(false);
      setIsTyping(false);

      // 다음 카드 공개 또는 총평
      if (cardIndex < 2) {
        setTimeout(() => {
          revealAndInterpretCard(cardIndex + 1, allSelectedCards);
        }, 1500);
      } else {
        // 3장 다 해석했으면 총평
        setTimeout(() => {
          giveFinalReading(allSelectedCards);
        }, 1500);
      }
    } catch (error) {
      console.error('해석 오류:', error);
      addMessage('assistant', '해석 중 오류가 발생했습니다.');
      setIsStreaming(false);
      setIsTyping(false);
    }
  };

  // 총평 + 보조덱 유도
  const giveFinalReading = async (allDrawnCards) => {
    setIsTyping(true);
    setIsStreaming(true);
    
    const cardDescriptions = allDrawnCards.map((card, idx) => {
      const position = idx === 0 ? '과거/현재' : idx === 1 ? '내면/감정' : '미래/결과';
      return `${position}: ${card.name}`;
    }).join('\n');

    const prompt = `당신은 친근하고 따뜻한 타로 상담가입니다.

내담자 고민: "${concern}"

뽑힌 카드:
${cardDescriptions}

세 카드의 흐름을 종합하여 총평을 해주세요.

요구사항:
- 친구에게 말하듯 자연스럽고 따뜻하게
- 150자 내외로 간결하게
- 희망적이고 긍정적으로 마무리
- "~네요", "~같아요" 사용
- 마지막에 "혹시 더 궁금한 부분이 있다면 보조덱을 뽑아볼까요?" 같은 유도 멘트 추가`;

    try {
      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key=${GEMINI_API_KEY}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }],
            generationConfig: {
              temperature: 0.9,
              topP: 0.95,
              topK: 40,
              maxOutputTokens: 250
            }
          })
        }
      );

      const data = await response.json();
      const fullText = data.candidates[0].content.parts[0].text;

      // 스트리밍 효과
      let currentText = '';
      for (let i = 0; i < fullText.length; i++) {
        currentText += fullText[i];
        setStreamingMessage(currentText);
        await new Promise(resolve => setTimeout(resolve, 20));
      }

      addMessage('assistant', fullText);
      setStreamingMessage('');
      setIsStreaming(false);
      setIsTyping(false);
      
      // 총평 완료!
      setFinalReadingComplete(true);
      
    } catch (error) {
      console.error('총평 오류:', error);
      addMessage('assistant', '총평 중 오류가 발생했습니다.');
      setIsStreaming(false);
      setIsTyping(false);
    }
  };

  const handleSubdeck = async () => {
    const shuffled = [...allCards].sort(() => Math.random() - 0.5);
    const usedCardIds = drawnCards.map(c => c.card_id);
    const availableCards = shuffled.filter(c => !usedCardIds.includes(c.card_id));
    const newCard = availableCards[0];

    if (!newCard) {
      alert('더 이상 뽑을 카드가 없습니다!');
      return;
    }

    setDrawnCards(prev => [...prev, newCard]);

    const cardNum = drawnCards.length + 1;
    addMessage('assistant', `추가 카드 ${cardNum - 3}번: ${newCard.name}`);

    setIsTyping(true);
    setIsStreaming(true);
    
    const prompt = `내담자의 고민: "${concern}"

기존에 뽑은 카드들이 있고, 추가로 이 카드가 나왔습니다:
${newCard.name}
키워드: ${newCard.keyword}
의미: ${newCard.meaning}

이 카드가 추가로 전하는 메시지를 간결하게 설명해주세요.

요구사항:
- 50자 내외로 짧게
- "~네요", "~같아요" 사용`;

    try {
      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key=${GEMINI_API_KEY}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }],
            generationConfig: {
              temperature: 0.9,
              topP: 0.95,
              topK: 40,
              maxOutputTokens: 100
            }
          })
        }
      );

      const data = await response.json();
      const fullText = data.candidates[0].content.parts[0].text;

      // 스트리밍 효과
      let currentText = '';
      for (let i = 0; i < fullText.length; i++) {
        currentText += fullText[i];
        setStreamingMessage(currentText);
        await new Promise(resolve => setTimeout(resolve, 20));
      }

      addMessage('assistant', fullText);
      setStreamingMessage('');
      
    } catch (error) {
      console.error('보조덱 오류:', error);
      addMessage('assistant', '해석 중 오류가 발생했습니다.');
    }
    
    setIsStreaming(false);
    setIsTyping(false);
  };

  const handleAdvice = async () => {
    setIsTyping(true);
    setIsStreaming(true);
    
    const cardDescriptions = drawnCards.map((card) => {
      return card.name;
    }).join(', ');

    const prompt = `내담자의 고민: "${concern}"
뽑힌 카드: ${cardDescriptions}

카드를 바탕으로 내담자에게 따뜻하고 실질적인 조언을 해주세요.

요구사항:
- 50자 내외로 간결하게
- 구체적인 행동 1가지
- "~네요", "~세요" 사용`;

    try {
      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key=${GEMINI_API_KEY}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }],
            generationConfig: {
              temperature: 0.9,
              topP: 0.95,
              topK: 40,
              maxOutputTokens: 100
            }
          })
        }
      );

      const data = await response.json();
      const fullText = data.candidates[0].content.parts[0].text;

      // 스트리밍 효과
      let currentText = '';
      for (let i = 0; i < fullText.length; i++) {
        currentText += fullText[i];
        setStreamingMessage(currentText);
        await new Promise(resolve => setTimeout(resolve, 20));
      }

      addMessage('assistant', fullText);
      setStreamingMessage('');
      
    } catch (error) {
      console.error('조언 오류:', error);
      addMessage('assistant', '조언 중 오류가 발생했습니다.');
    }
    
    setIsStreaming(false);
    setIsTyping(false);
  };

  const handleFortune = async () => {
    setIsTyping(true);
    setIsStreaming(true);
    
    const cardDescriptions = drawnCards.map((card) => {
      return card.name;
    }).join(', ');

    const prompt = `뽑힌 카드: ${cardDescriptions}

이 카드들을 바탕으로 운을 개선할 수 있는 개운법을 알려주세요.

요구사항:
- 50자 내외로 짧게
- 추천 색상 또는 행동 1가지만
- "~해보세요" 사용`;

    try {
      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key=${GEMINI_API_KEY}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }],
            generationConfig: {
              temperature: 0.9,
              topP: 0.95,
              topK: 40,
              maxOutputTokens: 100
            }
          })
        }
      );

      const data = await response.json();
      const fullText = data.candidates[0].content.parts[0].text;

      // 스트리밍 효과
      let currentText = '';
      for (let i = 0; i < fullText.length; i++) {
        currentText += fullText[i];
        setStreamingMessage(currentText);
        await new Promise(resolve => setTimeout(resolve, 20));
      }

      addMessage('assistant', fullText);
      setStreamingMessage('');
      
    } catch (error) {
      console.error('개운법 오류:', error);
      addMessage('assistant', '개운법 생성 중 오류가 발생했습니다.');
    }
    
    setIsStreaming(false);
    setIsTyping(false);
  };

  const handleEndConsultation = async () => {
    // 총조언 먼저
    setIsTyping(true);
    
    const cardsList = drawnCards.map(c => c.name).join(', ');
    
    const advicePrompt = `내담자 고민: "${concern}"
뽑힌 카드: ${cardsList}

상담을 마무리하며 내담자에게 따뜻한 격려와 실질적인 조언을 해주세요.

요구사항:
- 친구에게 말하듯 자연스럽게
- 100자 내외
- 희망적이고 긍정적으로
- "~네요", "~세요" 사용`;

    const finalAdvice = await callGeminiAPI(advicePrompt);
    addMessage('assistant', finalAdvice);
    setIsTyping(false);
    
    await new Promise(resolve => setTimeout(resolve, 1000));
    addMessage('assistant', '상담을 종료합니다. 좋은 하루 보내세요! 😊');

    try {
      const { data, error } = await supabase
        .from('consultations')
        .insert([{
          free_user_id: userId,
          free_user_name: userName,
          version_type: 'free',
          title: sessionTitle,
          concern: concern,
          cards_drawn: cardsList,
          created_at: new Date().toISOString()
        }])
        .select()
        .single();

      if (error) {
        console.error('저장 오류:', error);
      } else if (data) {
        setCurrentSessionId(data.id);
        console.log('상담 저장 완료:', data.id);
      }
    } catch (err) {
      console.error('저장 오류:', err);
    }

    setTimeout(() => {
      setStep('finished');
    }, 1500);
  };

  const handleShare = async () => {
    const cardsList = drawnCards.map(c => c.name).join(', ');
    
    // 전체 대화 내용 추출
    const conversationText = messages
      .filter(m => m.role === 'assistant')
      .map(m => m.content)
      .join('\n\n');

    const shareText = `🔮 타로 상담 결과

📝 고민: ${concern}

🃏 뽑힌 카드: ${cardsList}

💬 상담 내용:
${conversationText}

#타로 #타로상담 #만신카드`;

    // 모바일 공유 API 지원 확인
    if (navigator.share) {
      try {
        await navigator.share({
          title: '🔮 타로 상담 결과',
          text: shareText
        });
        console.log('공유 성공!');
      } catch (err) {
        if (err.name !== 'AbortError') {
          // 취소가 아닌 다른 에러면 클립보드로
          await navigator.clipboard.writeText(shareText);
          alert('클립보드에 복사되었습니다!');
        }
      }
    } else {
      // 공유 API 미지원 시 클립보드
      try {
        await navigator.clipboard.writeText(shareText);
        alert('전체 상담 내용이 클립보드에 복사되었습니다!\n원하는 곳에 붙여넣기 하세요.');
      } catch (err) {
        console.error('클립보드 복사 실패:', err);
        alert('공유 기능을 사용할 수 없습니다.');
      }
    }
  };

  const handleReset = () => {
    setStep('input');
    setConcern('');
    setSessionTitle('');
    setDisplayTitle('');
    setMessages([]);
    setDrawnCards([]);
    setStreamingMessage('');
    setCurrentSessionId(null);
    setCurrentCardIndex(0);
    setFinalReadingComplete(false);
  };

  const addMessage = (role, content) => {
    setMessages(prev => [...prev, { role, content }]);
  };

  if (step === 'loading') {
    return (
      <div style={{
        minHeight: '100vh',
        background: 'linear-gradient(135deg, #E0F7FA 0%, #B2EBF2 100%)',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        color: '#006064'
      }}>
        <div style={{ textAlign: 'center' }}>
          <div style={{ fontSize: '60px', marginBottom: '20px' }}>🔮</div>
          <div style={{ fontSize: '18px' }}>로딩중...</div>
          <div style={{ fontSize: '14px', marginTop: '10px', color: '#00838F' }}>
            만신카드 불러오는 중...
          </div>
        </div>
      </div>
    );
  }

  if (step === 'name_input') {
    return (
      <div style={{
        minHeight: '100vh',
        background: 'linear-gradient(135deg, #E0F7FA 0%, #B2EBF2 100%)',
        color: '#006064',
        padding: '30px 16px',
        display: 'flex',
        flexDirection: 'column',
        alignItems: 'center',
        justifyContent: 'center',
        fontFamily: 'system-ui, -apple-system, sans-serif'
      }}>
        <div style={{ maxWidth: '400px', width: '100%' }}>
          
          <div style={{
            position: 'fixed',
            top: '12px',
            right: '12px',
            background: 'linear-gradient(135deg, #00ACC1 0%, #0097A7 100%)',
            padding: '8px 16px',
            borderRadius: '20px',
            fontWeight: 'bold',
            fontSize: '13px',
            color: 'white',
            boxShadow: '0 4px 15px rgba(0, 172, 193, 0.4)',
            zIndex: 1000
          }}>
            무료판
          </div>

          <div style={{ textAlign: 'center', marginBottom: '40px' }}>
            <div style={{ fontSize: '80px', marginBottom: '20px' }}>🔮</div>
            <h1 style={{ fontSize: '28px', marginBottom: '12px', margin: 0 }}>
              환영합니다!
            </h1>
            <p style={{ color: '#00838F', fontSize: '15px', lineHeight: '1.6' }}>
              처음 오셨네요!<br/>
              타로 상담을 시작하기 전에<br/>
              이름을 알려주세요
            </p>
          </div>

          <div style={{
            background: 'white',
            borderRadius: '20px',
            padding: '30px',
            boxShadow: '0 10px 40px rgba(0, 172, 193, 0.2)'
          }}>
            <input
              type="text"
              value={tempName}
              onChange={(e) => setTempName(e.target.value)}
              placeholder="이름을 입력하세요"
              onKeyPress={(e) => e.key === 'Enter' && handleNameSubmit()}
              style={{
                width: '100%',
                padding: '16px',
                borderRadius: '12px',
                border: '2px solid #B2EBF2',
                background: '#E0F7FA',
                color: '#006064',
                fontSize: '16px',
                fontFamily: 'inherit',
                marginBottom: '16px',
                boxSizing: 'border-box',
                textAlign: 'center',
                fontWeight: 'bold'
              }}
            />
            
            <button
              onClick={handleNameSubmit}
              disabled={!tempName.trim()}
              style={{
                width: '100%',
                padding: '16px',
                borderRadius: '12px',
                border: 'none',
                background: tempName.trim() ? 'linear-gradient(135deg, #00ACC1 0%, #0097A7 100%)' : '#B2EBF2',
                color: 'white',
                fontSize: '17px',
                fontWeight: 'bold',
                cursor: tempName.trim() ? 'pointer' : 'not-allowed',
                transition: 'all 0.3s',
                boxShadow: tempName.trim() ? '0 4px 15px rgba(0, 172, 193, 0.3)' : 'none'
              }}
            >
              시작하기
            </button>
          </div>

          <div style={{
            marginTop: '30px',
            padding: '20px',
            background: 'white',
            borderRadius: '15px',
            textAlign: 'center',
            fontSize: '13px',
            color: '#00838F',
            boxShadow: '0 2px 10px rgba(0, 172, 193, 0.1)',
            lineHeight: '1.6'
          }}>
            이름은 안전하게 저장되며<br/>
            다음 방문 시 자동으로 불러옵니다
          </div>
        </div>
      </div>
    );
  }

  if (step === 'input') {
    return (
      <div style={{
        minHeight: '100vh',
        background: 'linear-gradient(135deg, #E0F7FA 0%, #B2EBF2 100%)',
        color: '#006064',
        padding: '16px',
        fontFamily: 'system-ui, -apple-system, sans-serif'
      }}>
        <div style={{ maxWidth: '500px', margin: '0 auto' }}>
          
          <div style={{
            position: 'fixed',
            top: '12px',
            right: '12px',
            background: 'linear-gradient(135deg, #00ACC1 0%, #0097A7 100%)',
            padding: '8px 16px',
            borderRadius: '20px',
            fontWeight: 'bold',
            fontSize: '13px',
            color: 'white',
            boxShadow: '0 4px 15px rgba(0, 172, 193, 0.4)',
            zIndex: 1000
          }}>
            무료판
          </div>

          <div style={{ textAlign: 'center', marginBottom: '30px', paddingTop: '50px' }}>
            <div style={{ fontSize: '60px', marginBottom: '16px' }}>🔮</div>
            <h1 style={{ fontSize: '26px', marginBottom: '8px', margin: '0 0 8px 0' }}>
              {visitCount === 1 ? (
                `${userName}님, 환영합니다!`
              ) : (
                `${userName}님, 다시 오셨네요!`
              )}
            </h1>
            <p style={{ color: '#00838F', fontSize: '14px', margin: 0 }}>
              {visitCount === 1 ? (
                '처음 오셨네요! 무료로 타로를 체험해보세요'
              ) : pastSessions.length > 0 ? (
                `${visitCount}번째 방문 • 지난번 ${pastSessions[0].title}`
              ) : (
                `${visitCount}번째 방문입니다`
              )}
            </p>
            {allCards.length > 0 && (
              <div style={{ fontSize: '12px', color: '#00ACC1', marginTop: '5px' }}>
                만신카드 {allCards.length}장 준비 완료
              </div>
            )}
          </div>

          <div style={{
            background: 'white',
            borderRadius: '16px',
            padding: '20px',
            boxShadow: '0 10px 40px rgba(0, 172, 193, 0.2)'
          }}>
            <textarea
              value={concern}
              onChange={(e) => setConcern(e.target.value)}
              placeholder="오늘은 어떤 고민이 있으신가요?"
              style={{
                width: '100%',
                minHeight: '100px',
                maxHeight: '180px',
                padding: '14px',
                borderRadius: '12px',
                border: '2px solid #B2EBF2',
                background: '#E0F7FA',
                color: '#006064',
                fontSize: '15px',
                resize: 'vertical',
                fontFamily: 'inherit',
                marginBottom: '14px',
                boxSizing: 'border-box'
              }}
            />
            
            <button
              onClick={handleStartConsultation}
              disabled={!concern.trim() || allCards.length === 0}
              style={{
                width: '100%',
                padding: '14px',
                borderRadius: '12px',
                border: 'none',
                background: (concern.trim() && allCards.length > 0) ? 'linear-gradient(135deg, #00ACC1 0%, #0097A7 100%)' : '#B2EBF2',
                color: 'white',
                fontSize: '16px',
                fontWeight: 'bold',
                cursor: (concern.trim() && allCards.length > 0) ? 'pointer' : 'not-allowed',
                transition: 'all 0.3s',
                boxShadow: (concern.trim() && allCards.length > 0) ? '0 4px 15px rgba(0, 172, 193, 0.3)' : 'none'
              }}
            >
              {allCards.length === 0 ? '만신카드 로딩 중...' : '상담 시작하기'}
            </button>
          </div>

          {pastSessions.length > 0 && (
            <div style={{ marginTop: '30px' }}>
              <h3 style={{ marginBottom: '16px', color: '#00838F', fontSize: '16px', margin: '0 0 16px 0' }}>
                과거 상담 내역
              </h3>
              {pastSessions.slice(0, 5).map((session) => (
                <div
                  key={session.id}
                  style={{
                    background: 'white',
                    padding: '14px',
                    borderRadius: '12px',
                    marginBottom: '12px',
                    boxShadow: '0 2px 10px rgba(0, 172, 193, 0.1)'
                  }}
                >
                  <div style={{ marginBottom: '8px', color: '#00ACC1', fontWeight: 'bold', fontSize: '13px' }}>
                    {new Date(session.created_at).toLocaleDateString()}
                  </div>
                  <div style={{ marginBottom: '8px', color: '#006064', fontSize: '14px' }}>{session.title}</div>
                  <div style={{ fontSize: '12px', color: '#00838F' }}>
                    {session.cards_drawn}
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>
      </div>
    );
  }

  if (step === 'consultation') {
    return (
      <div style={{
        height: '100vh',
        display: 'flex',
        flexDirection: 'column',
        background: 'linear-gradient(135deg, #E0F7FA 0%, #B2EBF2 100%)',
        color: '#006064'
      }}>
        <div style={{
          padding: '14px 16px',
          borderBottom: '2px solid #80DEEA',
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'space-between',
          background: 'white',
          boxShadow: '0 2px 10px rgba(0, 172, 193, 0.1)'
        }}>
          <button
            onClick={handleReset}
            style={{
              background: 'none',
              border: 'none',
              color: '#00ACC1',
              cursor: 'pointer',
              fontSize: '20px',
              padding: '4px'
            }}
          >
            <ArrowLeft />
          </button>
          <div style={{ flex: 1, textAlign: 'center', fontSize: '14px', padding: '0 12px', color: '#006064', fontWeight: 'bold' }}>
            {displayTitle || concern}
          </div>
          <div style={{ width: '28px' }}></div>
        </div>

        {drawnCards.length > 0 && (
          <div style={{
            padding: '12px 16px',
            borderBottom: '2px solid #80DEEA',
            background: 'white',
            overflowX: 'auto',
            whiteSpace: 'nowrap',
            boxShadow: '0 2px 10px rgba(0, 172, 193, 0.1)'
          }}>
            <div style={{ fontSize: '11px', color: '#00838F', marginBottom: '8px' }}>
              뽑힌 만신카드 ({drawnCards.length}장)
            </div>
            {drawnCards.map((card, idx) => (
              <span
                key={idx}
                style={{
                  display: 'inline-block',
                  padding: '6px 12px',
                  background: '#E0F7FA',
                  border: '2px solid #00ACC1',
                  borderRadius: '16px',
                  marginRight: '8px',
                  fontSize: '13px',
                  color: '#006064',
                  fontWeight: 'bold'
                }}
              >
                {card.name}
              </span>
            ))}
          </div>
        )}

        <div style={{
          flex: 1,
          overflowY: 'auto',
          padding: '16px'
        }}>
          {messages.map((msg, idx) => (
            <div
              key={idx}
              style={{
                marginBottom: '12px',
                display: 'flex',
                justifyContent: msg.role === 'user' ? 'flex-end' : 'flex-start'
              }}
            >
              <div style={{
                maxWidth: '85%',
                padding: '12px 16px',
                borderRadius: '16px',
                background: msg.role === 'user' ? '#00ACC1' : 'white',
                color: msg.role === 'user' ? 'white' : '#006064',
                whiteSpace: 'pre-wrap',
                wordBreak: 'break-word',
                boxShadow: '0 2px 10px rgba(0, 172, 193, 0.15)',
                fontSize: '14px',
                lineHeight: '1.5'
              }}>
                {msg.content}
              </div>
            </div>
          ))}

          {streamingMessage && (
            <div style={{
              marginBottom: '12px',
              display: 'flex',
              justifyContent: 'flex-start'
            }}>
              <div style={{
                maxWidth: '85%',
                padding: '12px 16px',
                borderRadius: '16px',
                background: 'white',
                color: '#006064',
                whiteSpace: 'pre-wrap',
                wordBreak: 'break-word',
                boxShadow: '0 2px 10px rgba(0, 172, 193, 0.15)',
                fontSize: '14px',
                lineHeight: '1.5'
              }}>
                {streamingMessage}
              </div>
            </div>
          )}

          {isTyping && !streamingMessage && (
            <div style={{ color: '#00838F', fontStyle: 'italic', fontSize: '13px' }}>
              입력 중...
            </div>
          )}

          <div ref={messagesEndRef} />
        </div>

        {finalReadingComplete && !isTyping && (
          <div style={{
            padding: '12px 16px',
            borderTop: '2px solid #80DEEA',
            display: 'flex',
            gap: '8px',
            flexWrap: 'wrap',
            background: 'white',
            boxShadow: '0 -2px 10px rgba(0, 172, 193, 0.1)'
          }}>
            <button
              onClick={handleSubdeck}
              style={{
                flex: 1,
                minWidth: '100px',
                padding: '12px',
                borderRadius: '10px',
                border: '2px solid #00ACC1',
                background: 'white',
                color: '#00ACC1',
                cursor: 'pointer',
                fontWeight: 'bold',
                transition: 'all 0.3s',
                fontSize: '13px'
              }}
            >
              보조덱
            </button>
            
            <button
              onClick={handleAdvice}
              style={{
                flex: 1,
                minWidth: '100px',
                padding: '12px',
                borderRadius: '10px',
                border: '2px solid #00ACC1',
                background: 'white',
                color: '#00ACC1',
                cursor: 'pointer',
                fontWeight: 'bold',
                transition: 'all 0.3s',
                fontSize: '13px'
              }}
            >
              조언
            </button>
            
            <button
              onClick={handleFortune}
              style={{
                flex: 1,
                minWidth: '100px',
                padding: '12px',
                borderRadius: '10px',
                border: '2px solid #00ACC1',
                background: 'white',
                color: '#00ACC1',
                cursor: 'pointer',
                fontWeight: 'bold',
                transition: 'all 0.3s',
                fontSize: '13px'
              }}
            >
              개운법
            </button>
            
            <button
              onClick={handleEndConsultation}
              style={{
                width: '100%',
                padding: '12px',
                borderRadius: '10px',
                border: 'none',
                background: 'linear-gradient(135deg, #00ACC1 0%, #0097A7 100%)',
                color: 'white',
                cursor: 'pointer',
                fontWeight: 'bold',
                boxShadow: '0 4px 15px rgba(0, 172, 193, 0.3)',
                fontSize: '14px'
              }}
            >
              상담 종료
            </button>
          </div>
        )}
      </div>
    );
  }

  if (step === 'finished') {
    return (
      <div style={{
        minHeight: '100vh',
        background: 'linear-gradient(135deg, #E0F7FA 0%, #B2EBF2 50%, #80DEEA 100%)',
        color: '#006064',
        padding: '30px 16px',
        display: 'flex',
        flexDirection: 'column',
        alignItems: 'center',
        justifyContent: 'center'
      }}>
        <div style={{ fontSize: '60px', marginBottom: '16px' }}>✨</div>
        <h1 style={{ fontSize: '26px', marginBottom: '16px', margin: '0 0 16px 0' }}>
          상담이 완료되었습니다!
        </h1>
        <p style={{ color: '#00838F', marginBottom: '30px', fontSize: '16px', margin: '0 0 30px 0' }}>
          좋은 하루 보내세요
        </p>
        
        <div style={{ display: 'flex', gap: '12px', flexDirection: 'column', width: '100%', maxWidth: '300px' }}>
          <button
            onClick={handleShare}
            style={{
              padding: '12px 24px',
              borderRadius: '10px',
              border: '2px solid #00ACC1',
              background: 'white',
              color: '#00ACC1',
              cursor: 'pointer',
              fontWeight: 'bold',
              fontSize: '15px',
              transition: 'all 0.3s'
            }}
          >
            공유하기
          </button>
          
          <button
            onClick={handleReset}
            style={{
              padding: '12px 24px',
              borderRadius: '10px',
              border: 'none',
              background: 'linear-gradient(135deg, #00ACC1 0%, #0097A7 100%)',
              color: 'white',
              cursor: 'pointer',
              fontWeight: 'bold',
              fontSize: '15px',
              boxShadow: '0 4px 15px rgba(0, 172, 193, 0.3)'
            }}
          >
            처음으로
          </button>
        </div>
      </div>
    );
  }

  return null;
}

export default App;
