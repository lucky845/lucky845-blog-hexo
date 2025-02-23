---
title: Java设计模式之享元模式详解
date: 2025-02-23 20:40:00
tags:
  - Java
  - 设计模式
categories:
  - 技术笔记
  - Java
abbrlink: 2f3e105f
---

## 什么是享元模式？

享元模式（Flyweight Pattern）是一种结构型设计模式，它通过共享来有效地支持大量细粒度对象的复用。享元模式尝试重用现有的同类对象，如果未找到匹配的对象，则创建新对象。

## 为什么使用享元模式？

1. 减少对象创建，节省内存空间
2. 提高系统性能
3. 实现对象的复用
4. 减少内存中对象的数量

## 实现示例

### 1. 基本实现

```java
// 享元接口
public interface Flyweight {
    void operation(String extrinsicState);
}

// 具体享元类
public class ConcreteFlyweight implements Flyweight {
    private String intrinsicState;
    
    public ConcreteFlyweight(String intrinsicState) {
        this.intrinsicState = intrinsicState;
    }
    
    @Override
    public void operation(String extrinsicState) {
        System.out.println("内部状态: " + intrinsicState);
        System.out.println("外部状态: " + extrinsicState);
    }
}

// 享元工厂
public class FlyweightFactory {
    private Map<String, Flyweight> flyweights = new HashMap<>();
    
    public Flyweight getFlyweight(String key) {
        if (!flyweights.containsKey(key)) {
            flyweights.put(key, new ConcreteFlyweight(key));
        }
        return flyweights.get(key);
    }
    
    public int getFlyweightCount() {
        return flyweights.size();
    }
}
```

### 2. 实际应用示例：字符编辑器

```java
// 字符享元类
public class CharacterFlyweight {
    private char character;
    private String font;
    private int size;
    
    public CharacterFlyweight(char character, String font, int size) {
        this.character = character;
        this.font = font;
        this.size = size;
    }
    
    public void display(int x, int y, String color) {
        System.out.printf("显示字符 %c (字体: %s, 大小: %d) 在位置(%d, %d)，颜色: %s%n",
                character, font, size, x, y, color);
    }
}

// 字符工厂
public class CharacterFactory {
    private Map<String, CharacterFlyweight> characters = new HashMap<>();
    
    public CharacterFlyweight getCharacter(char character, String font, int size) {
        String key = character + font + size;
        if (!characters.containsKey(key)) {
            characters.put(key, new CharacterFlyweight(character, font, size));
        }
        return characters.get(key);
    }
}

// 文本编辑器
public class TextEditor {
    private CharacterFactory factory;
    private List<CharacterFlyweight> characters;
    private List<Point> positions;
    private List<String> colors;
    
    public TextEditor() {
        factory = new CharacterFactory();
        characters = new ArrayList<>();
        positions = new ArrayList<>();
        colors = new ArrayList<>();
    }
    
    public void addCharacter(char c, int x, int y, String font, int size, String color) {
        CharacterFlyweight character = factory.getCharacter(c, font, size);
        characters.add(character);
        positions.add(new Point(x, y));
        colors.add(color);
    }
    
    public void display() {
        for (int i = 0; i < characters.size(); i++) {
            Point position = positions.get(i);
            characters.get(i).display(position.x, position.y, colors.get(i));
        }
    }
}
```

### 3. 棋盘游戏示例

```java
// 棋子享元类
public class ChessPieceFlyweight {
    private String type;    // 类型（兵、车、马等）
    private String color;   // 颜色（黑、白）
    private String image;   // 图片资源
    
    public ChessPieceFlyweight(String type, String color, String image) {
        this.type = type;
        this.color = color;
        this.image = image;
    }
    
    public void display(int x, int y) {
        System.out.printf("%s色%s显示在位置(%d, %d)，使用图片资源：%s%n",
                color, type, x, y, image);
    }
}

// 棋子工厂
public class ChessPieceFactory {
    private static Map<String, ChessPieceFlyweight> pieces = new HashMap<>();
    
    public static ChessPieceFlyweight getPiece(String type, String color) {
        String key = type + color;
        if (!pieces.containsKey(key)) {
            String image = "resources/" + color + "_" + type + ".png";
            pieces.put(key, new ChessPieceFlyweight(type, color, image));
        }
        return pieces.get(key);
    }
}

// 棋盘
public class ChessBoard {
    private ChessPieceFlyweight[][] board = new ChessPieceFlyweight[8][8];
    private int[][] positions = new int[8][8];
    
    public void placePiece(String type, String color, int x, int y) {
        ChessPieceFlyweight piece = ChessPieceFactory.getPiece(type, color);
        board[x][y] = piece;
        positions[x][y] = 1;
    }
    
    public void display() {
        for (int i = 0; i < 8; i++) {
            for (int j = 0; j < 8; j++) {
                if (positions[i][j] == 1) {
                    board[i][j].display(i, j);
                }
            }
        }
    }
}
```

### 4. 使用示例

```java
public class Client {
    public static void main(String[] args) {
        // 文本编辑器示例
        TextEditor editor = new TextEditor();
        editor.addCharacter('H', 0, 0, "Arial", 12, "black");
        editor.addCharacter('e', 10, 0, "Arial", 12, "black");
        editor.addCharacter('l', 20, 0, "Arial", 12, "black");
        editor.addCharacter('l', 30, 0, "Arial", 12, "black");
        editor.addCharacter('o', 40, 0, "Arial", 12, "black");
        editor.display();
        
        // 棋盘游戏示例
        ChessBoard board = new ChessBoard();
        board.placePiece("Pawn", "White", 1, 1);
        board.placePiece("Pawn", "White", 1, 2);
        board.placePiece("Knight", "Black", 0, 1);
        board.display();
    }
}
```

## 享元模式的优点

1. 大大减少对象的创建，降低系统的内存，使效率提高
2. 减少内存之外的其他资源占用
3. 实现了对象的复用
4. 系统更加简洁

## 享元模式的缺点

1. 使得系统变得复杂
2. 需要分离出外部状态和内部状态
3. 读取外部状态会使运行时间稍微变长

## 适用场景

1. 系统中有大量对象，这些对象消耗大量内存
2. 这些对象的状态大部分可以外部化
3. 这些对象可以按照内部状态分成很多组
4. 系统不依赖于这些对象的身份

## 与其他模式的关系

1. 组合模式：可以与享元模式一起使用
2. 单例模式：享元工厂通常是单例的
3. 状态模式：状态模式的对象可以共享

## 总结

享元模式是一种用于优化系统性能的设计模式，它通过共享对象来减少内存使用。在处理大量相似对象时，享元模式是一个很好的选择。但是，使用享元模式需要仔细考虑对象状态的划分，以及是否值得增加这种复杂性。

## 参考资料

- 《Design Patterns: Elements of Reusable Object-Oriented Software》
- 《Head First Design Patterns》
- Java API 文档
- Spring Framework 源码

---

希望这篇文章能帮助您更好地理解Java中的享元模式。如果您有任何问题，欢迎在评论区讨论！ 
abbrlink: '0'
---
 