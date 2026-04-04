```
import React, { useState, useCallback } from 'react';

interface CounterState {
  value: number;
  operation: string;
}

const CounterWithUndo: React.FC = () => {
  // 当前数字
  const [currentValue, setCurrentValue] = useState<number>(0);
  
  // 历史记录栈，存储每一步的状态
  const [history, setHistory] = useState<CounterState[]>([
    { value: 0, operation: '初始值' }
  ]);
  
  // 当前在历史记录中的位置
  const [historyIndex, setHistoryIndex] = useState<number>(0);

  // 添加新的状态到历史记录
  const addToHistory = useCallback((value: number, operation: string) => {
    const newState = { value, operation };
    // 如果当前不在历史记录的最新位置，需要截断后面的记录
    const newHistory = history.slice(0, historyIndex + 1);
    newHistory.push(newState);
    
    setHistory(newHistory);
    setHistoryIndex(newHistory.length - 1);
    setCurrentValue(value);
  }, [history, historyIndex]);

  // A按钮：数字加1
  const handleIncrement = useCallback(() => {
    const newValue = currentValue + 1;
    addToHistory(newValue, '+1');
  }, [currentValue, addToHistory]);

  // B按钮：数字减1
  const handleDecrement = useCallback(() => {
    const newValue = currentValue - 1;
    addToHistory(newValue, '-1');
  }, [currentValue, addToHistory]);

  // C按钮：撤销上一步操作
  const handleUndoOne = useCallback(() => {
    if (historyIndex > 0) {
      const newIndex = historyIndex - 1;
      const previousState = history[newIndex];
      setHistoryIndex(newIndex);
      setCurrentValue(previousState.value);
    }
  }, [history, historyIndex]);

  // D按钮：撤销上一步的上一步操作
  const handleUndoTwo = useCallback(() => {
    if (historyIndex > 1) {
      const newIndex = historyIndex - 2;
      const previousState = history[newIndex];
      setHistoryIndex(newIndex);
      setCurrentValue(previousState.value);
    }
  }, [history, historyIndex]);

  // 重做功能（额外功能）
  const handleRedo = useCallback(() => {
    if (historyIndex < history.length - 1) {
      const newIndex = historyIndex + 1;
      const nextState = history[newIndex];
      setHistoryIndex(newIndex);
      setCurrentValue(nextState.value);
    }
  }, [history, historyIndex]);

  const canUndoOne = historyIndex > 0;
  const canUndoTwo = historyIndex > 1;
  const canRedo = historyIndex < history.length - 1;

  return (
    <div style={{ 
      padding: '20px', 
      textAlign: 'center', 
      fontFamily: 'Arial, sans-serif',
      maxWidth: '500px',
      margin: '0 auto'
    }}>
      <h2>计数器 (支持撤销操作)</h2>
      
      {/* 当前数值显示 */}
      <div style={{ 
        fontSize: '48px', 
        fontWeight: 'bold', 
        margin: '20px 0',
        padding: '20px',
        border: '2px solid #ddd',
        borderRadius: '8px',
        backgroundColor: '#f9f9f9'
      }}>
        {currentValue}
      </div>

      {/* 主要操作按钮 */}
      <div style={{ marginBottom: '20px' }}>
        <button 
          onClick={handleIncrement}
          style={{
            fontSize: '18px',
            padding: '10px 20px',
            margin: '0 10px',
            backgroundColor: '#4CAF50',
            color: 'white',
            border: 'none',
            borderRadius: '5px',
            cursor: 'pointer'
          }}
        >
          A: +1
        </button>
        
        <button 
          onClick={handleDecrement}
          style={{
            fontSize: '18px',
            padding: '10px 20px',
            margin: '0 10px',
            backgroundColor: '#f44336',
            color: 'white',
            border: 'none',
            borderRadius: '5px',
            cursor: 'pointer'
          }}
        >
          B: -1
        </button>
      </div>

      {/* 撤销操作按钮 */}
      <div style={{ marginBottom: '20px' }}>
        <button 
          onClick={handleUndoOne}
          disabled={!canUndoOne}
          style={{
            fontSize: '16px',
            padding: '8px 16px',
            margin: '0 8px',
            backgroundColor: canUndoOne ? '#2196F3' : '#ccc',
            color: 'white',
            border: 'none',
            borderRadius: '5px',
            cursor: canUndoOne ? 'pointer' : 'not-allowed'
          }}
        >
          C: 撤销1步
        </button>
        
        <button 
          onClick={handleUndoTwo}
          disabled={!canUndoTwo}
          style={{
            fontSize: '16px',
            padding: '8px 16px',
            margin: '0 8px',
            backgroundColor: canUndoTwo ? '#9C27B0' : '#ccc',
            color: 'white',
            border: 'none',
            borderRadius: '5px',
            cursor: canUndoTwo ? 'pointer' : 'not-allowed'
          }}
        >
          D: 撤销2步
        </button>
        
        {/* 额外的重做按钮 */}
        <button 
          onClick={handleRedo}
          disabled={!canRedo}
          style={{
            fontSize: '16px',
            padding: '8px 16px',
            margin: '0 8px',
            backgroundColor: canRedo ? '#FF9800' : '#ccc',
            color: 'white',
            border: 'none',
            borderRadius: '5px',
            cursor: canRedo ? 'pointer' : 'not-allowed'
          }}
        >
          重做
        </button>
      </div>

      {/* 历史记录显示 */}
      <div style={{ 
        textAlign: 'left', 
        backgroundColor: '#f5f5f5', 
        padding: '15px', 
        borderRadius: '5px',
        marginTop: '20px'
      }}>
        <h4>操作历史:</h4>
        <div style={{ maxHeight: '200px', overflowY: 'auto' }}>
          {history.map((state, index) => (
            <div 
              key={index}
              style={{ 
                padding: '5px 10px',
                backgroundColor: index === historyIndex ? '#e3f2fd' : 'transparent',
                borderLeft: index === historyIndex ? '3px solid #2196F3' : 'none',
                marginBottom: '2px'
              }}
            >
              <strong>{index === historyIndex ? '→ ' : '  '}</strong>
              步骤 {index}: {state.operation} = {state.value}
            </div>
          ))}
        </div>
      </div>

      {/* 状态信息 */}
      <div style={{ 
        marginTop: '15px', 
        fontSize: '14px', 
        color: '#666',
        textAlign: 'left'
      }}>
        <p>当前位置: {historyIndex + 1} / {history.length}</p>
        <p>可撤销1步: {canUndoOne ? '是' : '否'}</p>
        <p>可撤销2步: {canUndoTwo ? '是' : '否'}</p>
        <p>可重做: {canRedo ? '是' : '否'}</p>
      </div>
    </div>
  );
};

export default CounterWithUndo;
```

##### 核心功能

四个主要按钮：

- A按钮: 数字加1 (+1)

- B按钮: 数字减1 (-1)

- C按钮: 撤销上一步操作

- D按钮: 撤销上一步的上一步操作

##### 技术实现

使用了以下React Hooks：

1. useState 管理三个关键状态：

- currentValue: 当前显示的数字

- history: 历史记录数组，存储每一步的操作

- historyIndex: 当前在历史记录中的位置

2. useCallback 优化事件处理函数，避免不必要的重渲染

##### 核心机制

- 历史记录管理: 每次操作都会保存到历史栈中

- 撤销逻辑: 通过移动历史索引来实现撤销功能

- 边界检查: 防止撤销操作越界

- 状态截断: 在历史中间进行新操作时，会清除后续的历史记录

##### 额外功能

- 可视化的操作历史显示

- 重做功能

- 按钮状态管理（自动禁用不可用的操作）

- 实时状态信息显示