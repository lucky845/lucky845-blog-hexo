---
title: Java设计模式之备忘录模式详解
date: 2025-02-23 21:50:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 9f0e805f
---

## 什么是备忘录模式？

备忘录模式（Memento Pattern）是一种行为型设计模式，它允许在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态，以便以后当需要时能将该对象恢复到原先保存的状态。

## 为什么使用备忘录模式？

1. 保存对象的历史状态
2. 实现撤销/重做功能
3. 不破坏对象的封装性
4. 提供状态恢复机制

## 实现示例

### 1. 基本实现

```java
// 备忘录类
public class Memento {
    private String state;
    
    public Memento(String state) {
        this.state = state;
    }
    
    public String getState() {
        return state;
    }
}

// 发起人类
public class Originator {
    private String state;
    
    public void setState(String state) {
        this.state = state;
    }
    
    public String getState() {
        return state;
    }
    
    public Memento saveStateToMemento() {
        return new Memento(state);
    }
    
    public void getStateFromMemento(Memento memento) {
        state = memento.getState();
    }
}

// 管理者类
public class Caretaker {
    private List<Memento> mementoList = new ArrayList<>();
    
    public void add(Memento state) {
        mementoList.add(state);
    }
    
    public Memento get(int index) {
        return mementoList.get(index);
    }
}
```

### 2. 文本编辑器示例

```java
// 文本编辑器状态
public class EditorState {
    private final String content;
    private final int cursorPosition;
    
    public EditorState(String content, int cursorPosition) {
        this.content = content;
        this.cursorPosition = cursorPosition;
    }
    
    public String getContent() {
        return content;
    }
    
    public int getCursorPosition() {
        return cursorPosition;
    }
}

// 文本编辑器
public class TextEditor {
    private String content;
    private int cursorPosition;
    
    public void write(String text) {
        content = (content == null ? "" : content) + text;
        cursorPosition = content.length();
    }
    
    public void setCursor(int position) {
        this.cursorPosition = position;
    }
    
    public EditorState save() {
        return new EditorState(content, cursorPosition);
    }
    
    public void restore(EditorState state) {
        this.content = state.getContent();
        this.cursorPosition = state.getCursorPosition();
    }
    
    public String getContent() {
        return content;
    }
    
    public int getCursorPosition() {
        return cursorPosition;
    }
}

// 编辑器历史记录
public class EditorHistory {
    private Stack<EditorState> history = new Stack<>();
    
    public void push(EditorState state) {
        history.push(state);
    }
    
    public EditorState pop() {
        if (!history.isEmpty()) {
            return history.pop();
        }
        return null;
    }
}
```

### 3. 游戏存档示例

```java
// 游戏状态
public class GameState {
    private final int level;
    private final int score;
    private final List<String> inventory;
    
    public GameState(int level, int score, List<String> inventory) {
        this.level = level;
        this.score = score;
        this.inventory = new ArrayList<>(inventory);
    }
    
    public int getLevel() {
        return level;
    }
    
    public int getScore() {
        return score;
    }
    
    public List<String> getInventory() {
        return new ArrayList<>(inventory);
    }
}

// 游戏
public class Game {
    private int level;
    private int score;
    private List<String> inventory = new ArrayList<>();
    
    public void play() {
        level++;
        score += 100;
        inventory.add("Item" + level);
    }
    
    public GameState save() {
        return new GameState(level, score, inventory);
    }
    
    public void restore(GameState state) {
        this.level = state.getLevel();
        this.score = state.getScore();
        this.inventory = state.getInventory();
    }
    
    public String getStatus() {
        return String.format("Level: %d, Score: %d, Items: %s", 
                level, score, inventory);
    }
}

// 游戏存档管理
public class GameSaveManager {
    private Map<String, GameState> saves = new HashMap<>();
    
    public void saveGame(String name, GameState state) {
        saves.put(name, state);
    }
    
    public GameState loadGame(String name) {
        return saves.get(name);
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 基本示例
        Originator originator = new Originator();
        Caretaker caretaker = new Caretaker();
        
        originator.setState("State #1");
        caretaker.add(originator.saveStateToMemento());
        
        originator.setState("State #2");
        caretaker.add(originator.saveStateToMemento());
        
        originator.setState("State #3");
        System.out.println("Current State: " + originator.getState());
        
        originator.getStateFromMemento(caretaker.get(1));
        System.out.println("First saved State: " + originator.getState());
        
        // 文本编辑器示例
        TextEditor editor = new TextEditor();
        EditorHistory history = new EditorHistory();
        
        editor.write("Hello");
        history.push(editor.save());
        
        editor.write(" World");
        history.push(editor.save());
        
        editor.write("!");
        System.out.println("Current text: " + editor.getContent());
        
        editor.restore(history.pop());
        System.out.println("After undo: " + editor.getContent());
        
        // 游戏存档示例
        Game game = new Game();
        GameSaveManager saveManager = new GameSaveManager();
        
        game.play();
        saveManager.saveGame("save1", game.save());
        
        game.play();
        System.out.println("Current status: " + game.getStatus());
        
        game.restore(saveManager.loadGame("save1"));
        System.out.println("After loading save1: " + game.getStatus());
    }
}
```

## 备忘录模式的优点

1. 提供了状态恢复的机制
2. 不破坏对象的封装性
3. 提供了可靠的实现方式
4. 简化了发起人（Originator）的实现

## 备忘录模式的缺点

1. 可能会消耗大量的内存
2. 可能需要管理大量的备忘录对象
3. 可能需要完整存储对象的状态，影响性能

## 适用场景

1. 需要保存和恢复对象的状态
2. 需要实现撤销/重做功能
3. 直接访问对象的状态会破坏其封装性
4. 需要提供回滚操作

## 与其他模式的关系

1. 命令模式：可以使用备忘录模式来实现命令的撤销功能
2. 原型模式：可以使用原型模式来实现备忘录
3. 迭代器模式：可以使用迭代器来访问备忘录的历史记录

## 总结

备忘录模式是一种用于保存和恢复对象状态的设计模式。它在不破坏封装性的前提下，提供了一种可靠的状态恢复机制。这种模式在需要实现撤销/重做功能，或者需要保存对象历史状态的场景中特别有用，如文本编辑器、游戏存档等。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Java API 文档
- Spring Framework 源码

---

希望这篇文章能帮助您更好地理解Java中的备忘录模式。如果您有任何问题，欢迎在评论区讨论！ 
abbrlink: '0'
---
 