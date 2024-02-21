# 德州扑克

1. 选择起手范围
2. 控制底池大小
3. 控制对手手牌率实现

下注原则：价值下注或者诈唬下注

阻挡牌：

当阻挡了对手的诈唬组合，对手的下注多半代表牌力，此时应放弃抓诈。

成牌转诈唬需要满足几个条件：

1. 对手有一定牌力，但不是顶端范围。
2. 对手知道我们击中了牌面，且认为我们的范围很强。
3. 我们的牌力不够强，，但是阻挡了对手的一些顶端范围。
4. 对手在面对强牌时，有较高的弃牌率。

此时可以将成牌转成诈唬，扮演范围中的强牌而诈唬走对手的中等牌。

薄价值下注：当对手认为我们诈唬可能性很高时，可以用比对方略强一点的牌扮演诈唬牌，让对方来抓。

## 胜率

### 底牌胜率

1. 大对子 vs 小对子：80%
2. 主宰踢脚：70% ~ 75%
3. 两大 vs 两小：60% ~ 65%
4. 一大一小 vs 两中间：56% ~ 58%
5. 一大一中 vs 一中一小：62% ~ 64%
6. 大对子 vs 两小：80% ~ 90%
7. 中对子 vs 一大一小：70%
8. 小对子 vs 两大：50% ~ 55%

### 翻拍前全压

AQs ≈ TT，AQo ≈ 99， AJs ≈ 88，AJo ≈ 77，ATs ≈ 66，ATo ≈ 55。

### 投机翻牌圈

1. 口袋对中三条：13.3%
2. 天同花：0.85%
3. 前门听花：13.4%
4. 天顺：1.3%
5. 两端听顺：10.48%
6. AK中对子：32.43%
7. JJ发高张：52%
8. Ax中A：17.3%

## 翻牌前的再加注

3bet策略：

1. 隔离鱼
2. 扮演强牌
3. 平衡范围
4. 对手弃牌过多

跟注4bet：

小同花连牌 > A同花 > 对子 > 中高牌
