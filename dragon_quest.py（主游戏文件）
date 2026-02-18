"""
DeepSeek AI 驱动的勇者斗恶龙文本游戏
新手友好版 - 无需配置即可运行
"""

import json
import os
import random
import time
from dataclasses import dataclass, asdict
from typing import Dict, List, Optional
import sys

# 尝试导入requests，如果没有就使用本地模式
try:
    import requests
    HAS_REQUESTS = True
except ImportError:
    HAS_REQUESTS = False
    print("提示：未安装 requests 库，将以本地模式运行")
    print("如需AI功能，请运行：pip install requests\n")

@dataclass
class Player:
    name: str = "勇者"
    level: int = 1
    hp: int = 30
    max_hp: int = 30
    mp: int = 10
    max_mp: int = 10
    attack: int = 8
    defense: int = 5
    speed: int = 5
    exp: int = 0
    next_level_exp: int = 20
    gold: int = 50
    inventory: List[Dict] = None
    spells: List[str] = None
    location: str = "拉达托姆城"
    
    def __post_init__(self):
        if self.inventory is None:
            self.inventory = [
                {"name": "药草", "type": "heal", "effect": 15, "count": 3},
                {"name": "火把", "type": "tool", "count": 5}
            ]
        if self.spells is None:
            self.spells = []

@dataclass
class Monster:
    name: str
    hp: int
    max_hp: int
    attack: int
    defense: int
    speed: int
    exp_reward: int
    gold_reward: int
    description: str
    weakness: Optional[str] = None

class DQWorld:
    """勇者斗恶龙经典世界观"""
    
    LOCATIONS = {
        "拉达托姆城": {
            "description": "被邪恶笼罩的王国首都，龙王盘踞在西北角的黑暗城堡",
            "connections": ["草原", "道具店", "武器店", "城堡"],
            "danger_level": 0,
            "detail": "城堡高耸入云，但西北角被黑云笼罩。村民们面带忧色地交谈着。"
        },
        "草原": {
            "description": "拉达托姆城外的广阔草原，史莱姆和蜻蜓怪出没",
            "connections": ["拉达托姆城", "洞穴", "沼泽"],
            "danger_level": 1,
            "monsters": ["史莱姆", "蜻蜓怪", "顽皮鼹鼠"],
            "detail": "风吹草低，远处有奇怪的蠕动声。一朵蓝色的史莱姆正在晒太阳。"
        },
        "洞穴": {
            "description": "阴暗潮湿的地下洞穴，传说深处有古代宝藏",
            "connections": ["草原", "地下湖"],
            "danger_level": 2,
            "monsters": ["蝙蝠", "骸骨", "食人魔"],
            "detail": "水滴从钟乳石落下，回声阵阵。深处传来骨骼摩擦的声响..."
        },
        "沼泽": {
            "description": "毒气弥漫的湿地，强力的怪物在此栖息",
            "connections": ["草原", "龙王城"],
            "danger_level": 3,
            "monsters": ["毒蝎", "沼泽怪", "魔法师"],
            "detail": "紫色的雾气缭绕，脚下的土地松软危险。远处城堡的轮廓若隐若现。"
        },
        "龙王城": {
            "description": "龙王的黑暗要塞，世界恐惧的源头",
            "connections": ["沼泽"],
            "danger_level": 5,
            "monsters": ["恶魔骑士", "龙王"],
            "detail": "黑色的巨石堆砌而成，天空被乌云遮蔽。恐怖的威压让人呼吸困难。"
        },
        "道具店": {
            "description": "售卖各种冒险道具的商店",
            "connections": ["拉达托姆城"],
            "danger_level": 0,
            "shop": True,
            "detail": "货架上摆满了药草、解毒草和奇奇怪怪的小玩意。老板笑眯眯地看着你。"
        },
        "武器店": {
            "description": "锻造武器和防具的店铺",
            "connections": ["拉达托姆城"],
            "danger_level": 0,
            "shop": True,
            "detail": "铁匠的锤子声叮当作响，墙上挂着剑、斧头和闪亮的铠甲。"
        },
        "城堡": {
            "description": "拉达托姆王的居城",
            "connections": ["拉达托姆城"],
            "danger_level": 0,
            "detail": "大理石地面光可鉴人，卫兵肃立两旁。国王坐在远处的王座上。"
        }
    }
    
    MONSTERS = {
        "史莱姆": Monster("史莱姆", 8, 8, 5, 3, 2, 2, 3, 
                         "蓝色的果冻状生物，最弱的怪物", "火"),
        "蜻蜓怪": Monster("蜻蜓怪", 12, 12, 7, 4, 8, 4, 5,
                         "巨大的蜻蜓，速度极快"),
        "顽皮鼹鼠": Monster("顽皮鼹鼠", 10, 10, 6, 5, 4, 3, 4,
                          "喜欢恶作剧的地下生物"),
        "蝙蝠": Monster("蝙蝠", 10, 10, 6, 2, 10, 3, 4,
                       "洞窟中群居的吸血蝙蝠"),
        "骸骨": Monster("骸骨", 18, 18, 12, 8, 4, 8, 10,
                       "复活的骷髅战士", "火"),
        "食人魔": Monster("食人魔", 25, 25, 15, 6, 5, 15, 20,
                        "丑陋而强壮的洞穴怪物"),
        "毒蝎": Monster("毒蝎", 20, 20, 12, 8, 6, 12, 15,
                       "尾巴带剧毒的蝎子"),
        "沼泽怪": Monster("沼泽怪", 30, 30, 18, 12, 3, 20, 25,
                        "由沼泽淤泥形成的怪物"),
        "魔法师": Monster("魔法师", 22, 22, 8, 5, 8, 18, 30,
                        "堕落的宫廷魔法师，会使用咒语"),
        "恶魔骑士": Monster("恶魔骑士", 50, 50, 25, 20, 12, 50, 100,
                          "龙王的亲信，全身被黑甲覆盖"),
        "龙王": Monster("龙王", 200, 200, 50, 30, 20, 0, 0,
                      "世界的灾厄之源，拥有毁灭性的火焰吐息")
    }
    
    SHOP_ITEMS = {
        "药草": {"price": 10, "type": "heal", "effect": 15, "desc": "恢复15点HP"},
        "上等药草": {"price": 50, "type": "heal", "effect": 50, "desc": "恢复50点HP"},
        "解毒草": {"price": 20, "type": "cure", "desc": "解除中毒状态"},
        "火把": {"price": 5, "type": "tool", "desc": "照亮黑暗"},
        "铁剑": {"price": 100, "type": "weapon", "attack": 5, "desc": "攻击+5"},
        "皮甲": {"price": 80, "type": "armor", "defense": 4, "desc": "防御+4"}
    }

class SimpleNarrator:
    """本地叙事生成器（无需AI）"""
    
    BATTLE_DESCRIPTIONS = {
        "攻击": [
            "你挥剑斩向敌人，剑光闪过！",
            "你大喝一声，全力一击！",
            "你灵活地绕到侧面发起攻击！",
            "你瞄准破绽，迅猛出击！"
        ],
        "受伤": [
            "敌人反击，你勉强避开要害！",
            "你受了伤，但咬紧牙关坚持！",
            "攻击打在你的护甲上，震得你生疼！",
            "你踉跄了一下，重新站稳阵脚！"
        ],
        "胜利": [
            "敌人倒下了，你获得了胜利！",
            "随着最后一击，战斗结束了！",
            "你喘着气，看着敌人的残骸！"
        ]
    }
    
    def describe_scene(self, location: str, world: DQWorld) -> str:
        """描述场景"""
        loc = world.LOCATIONS[location]
        return f"\n📍 {location}\n   {loc['description']}\n   {loc.get('detail', '')}"
    
    def describe_battle(self, action: str, damage: int, is_player: bool = True) -> str:
        """描述战斗"""
        if is_player:
            desc = random.choice(self.BATTLE_DESCRIPTIONS["攻击"])
            return f"{desc} 造成 {damage} 点伤害！"
        else:
            desc = random.choice(self.BATTLE_DESCRIPTIONS["受伤"])
            return f"{desc} 受到 {damage} 点伤害！"

class DeepSeekNarrator(SimpleNarrator):
    """DeepSeek AI 叙事生成器"""
    
    def __init__(self, api_key: str):
        super().__init__()
        self.api_key = api_key
        self.api_url = "https://api.deepseek.com/v1/chat/completions"
        self.enabled = True
        
    def _call_api(self, prompt: str, max_tokens: int = 150) -> str:
        """调用API"""
        if not self.enabled:
            return None
            
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json"
        }
        
        payload = {
            "model": "deepseek-chat",
            "messages": [
                {"role": "system", "content": "你是《勇者斗恶龙》世界的叙事大师，风格温暖幽默，像鸟山明的画风。保持简洁（80字内）。"},
                {"role": "user", "content": prompt}
            ],
            "temperature": 0.8,
            "max_tokens": max_tokens
        }
        
        try:
            response = requests.post(
                self.api_url, 
                headers=headers, 
                json=payload,
                timeout=10
            )
            response.raise_for_status()
            return response.json()["choices"][0]["message"]["content"]
        except Exception as e:
            print(f"[AI服务暂时不可用，切换本地模式: {str(e)}]")
            self.enabled = False
            return None
    
    def describe_scene(self, location: str, world: DQWorld) -> str:
        """AI描述场景"""
        if not self.enabled:
            return super().describe_scene(location, world)
            
        prompt = f"描述勇者斗恶龙中的'{location}'场景：{world.LOCATIONS[location]['description']}。加入鸟山明风格的视觉细节和氛围。"
        ai_desc = self._call_api(prompt)
        
        if ai_desc:
            return f"\n📍 {location}\n🎭 {ai_desc}"
        else:
            return super().describe_scene(location, world)

class GameEngine:
    """游戏主引擎"""
    
    def __init__(self):
        self.player = Player()
        self.world = DQWorld()
        self.narrator = None
        self.state = "explore"
        self.current_monster: Optional[Monster] = None
        self.turn_count = 0
        self.running = True
        
        # 检查API密钥
        api_key = os.getenv("DEEPSEEK_API_KEY")
        if api_key and HAS_REQUESTS:
            print("✨ AI叙事模式已启用！")
            self.narrator = DeepSeekNarrator(api_key)
        else:
            print("🎮 本地叙事模式（输入 'ai' 查看如何开启智能叙事）")
            self.narrator = SimpleNarrator()
        
        self.commands = {
            "移动": self.cmd_move,
            "m": self.cmd_move,
            "探索": self.cmd_explore,
            "e": self.cmd_explore,
            "状态": self.cmd_status,
            "s": self.cmd_status,
            "物品": self.cmd_inventory,
            "i": self.cmd_inventory,
            "魔法": self.cmd_magic,
            "存档": self.cmd_save,
            "读取": self.cmd_load,
            "帮助": self.cmd_help,
            "h": self.cmd_help,
            "ai": self.cmd_ai_info,
            "商店": self.cmd_shop,
            "talk": self.cmd_talk,
            "休息": self.cmd_rest,
            "quit": self.cmd_quit,
            "退出": self.cmd_quit
        }
    
    def start(self):
        """游戏开始"""
        self._print_title()
        self._setup_player()
        self.main_loop()
    
    def _print_title(self):
        print("""
    ╔══════════════════════════════════════╗
    ║     DRAGON QUEST: TEXT ADVENTURE     ║
    ║          勇者斗恶龙：文本传说          ║
    ╠══════════════════════════════════════╣
    ║  指令：移动 探索 状态 物品 魔法 帮助    ║
    ╚══════════════════════════════════════╝
        """)
    
    def _setup_player(self):
        print("👤 创建你的勇者")
        name = input("请输入名字（直接回车叫'勇者'）：").strip()
        if name:
            self.player.name = name
        
        print(f"\n🎭 背景故事：")
        print(f"   {self.player.name}，拉达托姆王国的希望。")
        print(f"   龙王掳走了公主，黑暗笼罩大地...")
        print(f"   你站在城堡门前，握紧木剑。\n")
        input("按回车开始冒险...")
    
    def main_loop(self):
        """主循环"""
        while self.running:
            self.turn_count += 1
            
            if self.state == "explore":
                self.explore_phase()
            elif self.state == "battle":
                self.battle_phase()
            elif self.state == "shop":
                self.shop_phase()
            
            # 检查死亡
            if self.player.hp <= 0:
                self.game_over()
                break
    
    def explore_phase(self):
        """探索阶段"""
        # 显示场景
        desc = self.narrator.describe_scene(self.player.location, self.world)
        print(desc)
        
        loc = self.world.LOCATIONS[self.player.location]
        
        # 显示选项
        print(f"\n🗺️  可前往：{' | '.join(loc['connections'])}")
        print("💡 提示：输入'帮助'查看所有指令")
        
        cmd = input("\n> ").strip().lower()
        
        # 解析指令
        parts = cmd.split(maxsplit=1)
        action = parts[0]
        arg = parts[1] if len(parts) > 1 else ""
        
        if action in self.commands:
            self.commands[action](arg)
        else:
            print("❓ 未知指令，输入'帮助'查看列表")
    
    def cmd_move(self, destination: str):
        """移动"""
        if not destination:
            dest = input("去哪里？> ").strip()
        else:
            dest = destination
            
        current = self.world.LOCATIONS[self.player.location]
        
        if dest in current['connections']:
            self.player.location = dest
            print(f"\n🚶 你来到了{dest}...")
            
            # 遇敌判定
            new_loc = self.world.LOCATIONS[dest]
            if new_loc['danger_level'] > 0 and random.random() < 0.4:
                self.trigger_battle(new_loc)
            elif new_loc.get('shop'):
                self.state = "shop"
        else:
            print(f"❌ 无法从{self.player.location}直接到达{dest}")
            print(f"   可前往：{' | '.join(current['connections'])}")
    
    def cmd_explore(self, _):
        """探索"""
        loc = self.world.LOCATIONS[self.player.location]
        
        events = [
            ("你发现了一些金币！", "gold", random.randint(5, 20)),
            ("地上有奇怪的脚印...", "clue", None),
            ("风吹过，带来远方的气息", "nothing", None),
            ("这里似乎没有什么特别的", "nothing", None),
            ("你发现了隐藏的药草！", "item", {"name": "药草", "type": "heal", "effect": 15, "count": 1})
        ]
        
        event = random.choice(events)
        print(f"\n🔍 {event[0]}")
        
        if event[1] == "gold":
            self.player.gold += event[2]
            print(f"   获得 {event[2]} G！")
        elif event[1] == "item":
            self.add_item(event[2])
        
        # 遇敌
        if loc['danger_level'] > 0 and random.random() < 0.3:
            self.trigger_battle(loc)
    
    def trigger_battle(self, location_data: Dict):
        """触发战斗"""
        monster_name = random.choice(location_data['monsters'])
        template = self.world.MONSTERS[monster_name]
        
        # 复制怪物
        self.current_monster = Monster(
            name=template.name,
            hp=template.hp,
            max_hp=template.max_hp,
            attack=template.attack,
            defense=template.defense,
            speed=template.speed,
            exp_reward=template.exp_reward,
            gold_reward=template.gold_reward,
            description=template.description,
            weakness=template.weakness
        )
        
        print(f"\n⚔️  遭遇战！{self.current_monster.name} 出现了！")
        print(f"   {self.current_monster.description}")
        self.state = "battle"
    
    def battle_phase(self):
        """战斗阶段"""
        m = self.current_monster
        
        print(f"\n{'='*40}")
        print(f"🧑 {self.player.name:<8} HP:{self.player.hp:>3}/{self.player.max_hp:<3} MP:{self.player.mp:>2}/{self.player.max_mp:<2}")
        print(f"👹 {m.name:<8} HP:{m.hp:>3}/{m.max_hp:<3}")
        print(f"{'='*40}")
        
        print("\n[攻击] [魔法] [物品] [逃跑]")
        cmd = input("> ").strip().lower()
        
        if cmd in ["攻击", "a", "1"]:
            self.battle_attack()
        elif cmd in ["魔法", "m", "2"]:
            self.battle_magic()
        elif cmd in ["物品", "i", "3"]:
            self.battle_item()
        elif cmd in ["逃跑", "r", "4"]:
            self.battle_flee()
        else:
            print("无效指令")
    
    def battle_attack(self):
        """普通攻击"""
        m = self.current_monster
        
        # 计算伤害
        damage = max(1, self.player.attack - m.defense // 2)
        damage = int(damage * random.uniform(0.9, 1.1))
        
        m.hp -= damage
        
        # 叙事
        desc = self.narrator.describe_battle("攻击", damage, True)
        print(f"\n{desc}")
        
        if m.hp <= 0:
            self.win_battle()
        else:
            self.monster_turn()
    
    def battle_magic(self):
        """使用魔法"""
        if not self.player.spells:
            print("你还不会任何魔法！")
            return
        
        print("魔法列表：")
        for i, spell in enumerate(self.player.spells, 1):
            cost = 4 if spell == "霍伊米" else 6
            print(f"{i}. {spell} (MP{cost})")
        
        choice = input("选择（0取消）：").strip()
        if choice == "0":
            return
        
        try:
            idx = int(choice) - 1
            spell = self.player.spells[idx]
            cost = 4 if spell == "霍伊米" else 6
            
            if self.player.mp < cost:
                print("MP不足！")
                return
            
            self.player.mp -= cost
            
            if spell == "霍伊米":
                heal = 25
                self.player.hp = min(self.player.max_hp, self.player.hp + heal)
                print(f"使用了霍伊米！恢复 {heal} HP！")
            elif spell == "吉拉":
                damage = 15
                self.current_monster.hp -= damage
                print(f"使用了吉拉！造成 {damage} 伤害！")
            
            if self.current_monster.hp <= 0:
                self.win_battle()
            else:
                self.monster_turn()
        except (ValueError, IndexError):
            print("无效选择")
    
    def battle_item(self):
        """使用物品"""
        heals = [i for i in self.player.inventory if i['type'] == 'heal']
        
        if not heals:
            print("没有可用的恢复道具！")
            return
        
        print("可用道具：")
        for i, item in enumerate(heals, 1):
            print(f"{i}. {item['name']} x{item['count']}")
        
        try:
            idx = int(input("选择（0取消）：")) - 1
            if idx == -1:
                return
            
            item = heals[idx]
            heal = item['effect']
            self.player.hp = min(self.player.max_hp, self.player.hp + heal)
            item['count'] -= 1
            
            # 清理用完的道具
            if item['count'] <= 0:
                self.player.inventory.remove(item)
            
            print(f"使用了{item['name']}，恢复{heal}HP！")
            self.monster_turn()
        except (ValueError, IndexError):
            print("无效选择")
    
    def battle_flee(self):
        """逃跑"""
        if random.random() < 0.6:
            print("成功逃跑了！")
            self.state = "explore"
            self.current_monster = None
        else:
            print("逃跑失败！")
            self.monster_turn()
    
    def monster_turn(self):
        """怪物回合"""
        m = self.current_monster
        
        damage = max(1, m.attack - self.player.defense // 2)
        damage = int(damage * random.uniform(0.8, 1.2))
        
        self.player.hp -= damage
        
        desc = self.narrator.describe_battle("受伤", damage, False)
        print(f"{m.name} {desc}")
        
        if self.player.hp <= 0:
            print("\n💀 你被击败了...")
    
    def win_battle(self):
        """胜利"""
        m = self.current_monster
        
        print(f"\n✨ 击败了 {m.name}！")
        print(f"获得 {m.exp_reward} EXP，{m.gold_reward} G！")
        
        self.player.exp += m.exp_reward
        self.player.gold += m.gold_reward
        
        # 升级检查
        while self.player.exp >= self.player.next_level_exp:
            self.level_up()
        
        self.state = "explore"
        self.current_monster = None
    
    def level_up(self):
        """升级"""
        self.player.level += 1
        self.player.exp -= self.player.next_level_exp
        self.player.next_level_exp = int(self.player.next_level_exp * 1.5)
        
        # 成长
        hp_up = random.randint(3, 6)
        mp_up = random.randint(1, 3)
        
        self.player.max_hp += hp_up
        self.player.hp = self.player.max_hp
        self.player.max_mp += mp_up
        self.player.mp = self.player.max_mp
        self.player.attack += random.randint(1, 3)
        self.player.defense += random.randint(1, 2)
        
        print(f"\n🆙 升级！Lv.{self.player.level}！")
        print(f"   HP+{hp_up} MP+{mp_up} 其他属性提升！")
        
        # 学魔法
        if self.player.level == 3 and "霍伊米" not in self.player.spells:
            self.player.spells.append("霍伊米")
            print("   学会了 霍伊米（恢复魔法）！")
        elif self.player.level == 5 and "吉拉" not in self.player.spells:
            self.player.spells.append("吉拉")
            print("   学会了 吉拉（攻击魔法）！")
    
    def shop_phase(self):
        """商店界面"""
        print(f"\n🏪 欢迎来到商店！持有金币：{self.player.gold}G")
        print("商品列表：")
        
        items = list(self.world.SHOP_ITEMS.items())
        for i, (name, data) in enumerate(items, 1):
            print(f"{i}. {name} - {data['price']}G ({data['desc']})")
        print("0. 离开商店")
        
        try:
            choice = int(input("购买（输入编号）："))
            if choice == 0:
                self.state = "explore"
                self.player.location = "拉达托姆城"
                return
            
            item_name, item_data = items[choice - 1]
            
            if self.player.gold < item_data['price']:
                print("金币不足！")
                return
            
            self.player.gold -= item_data['price']
            
            # 处理装备
            if item_data['type'] in ['weapon', 'armor']:
                if item_data['type'] == 'weapon':
                    self.player.attack += item_data.get('attack', 0)
                else:
                    self.player.defense += item_data.get('defense', 0)
                print(f"装备了 {item_name}！")
            else:
                # 道具
                new_item = {
                    "name": item_name,
                    "type": item_data['type'],
                    "count": 1
                }
                if 'effect' in item_data:
                    new_item['effect'] = item_data['effect']
                self.add_item(new_item)
                print(f"购买了 {item_name}！")
                
        except (ValueError, IndexError):
            print("无效选择")
    
    def add_item(self, item):
        """添加物品到背包"""
        # 检查是否已有
        for inv in self.player.inventory:
            if inv['name'] == item['name'] and inv['type'] == item['type']:
                inv['count'] += item['count']
                return
        
        self.player.inventory.append(item)
    
    def cmd_status(self, _):
        """状态"""
        print(f"\n{'='*30}")
        print(f"  {self.player.name}  Lv.{self.player.level}")
        print(f"{'='*30}")
        print(f"❤️  HP: {self.player.hp}/{self.player.max_hp}")
        print(f"🔮 MP: {self.player.mp}/{self.player.max_mp}")
        print(f"⚔️  攻击: {self.player.attack}  防御: {self.player.defense}")
        print(f"💨 速度: {self.player.speed}")
        print(f"⭐ EXP: {self.player.exp}/{self.player.next_level_exp}")
        print(f"💰 金钱: {self.player.gold} G")
        print(f"📍 位置: {self.player.location}")
    
    def cmd_inventory(self, _):
        """背包"""
        if not self.player.inventory:
            print("背包是空的")
            return
        
        print("\n🎒 背包：")
        for item in self.player.inventory:
            print(f"   {item['name']} x{item['count']}")
    
    def cmd_magic(self, _):
        """魔法"""
        if not self.player.spells:
            print("尚未学会魔法（Lv.3学会霍伊米）")
            return
        
        print("\n✨ 已学会：")
        for spell in self.player.spells:
            cost = 4 if spell == "霍伊米" else 6
            print(f"   {spell} (消耗MP{cost})")
    
    def cmd_save(self, _):
        """存档"""
        save_data = {
            "player": asdict(self.player),
            "turn": self.turn_count
        }
        
        filename = f"save_{self.player.name}.json"
        with open(filename, 'w', encoding='utf-8') as f:
            json.dump(save_data, f, ensure_ascii=False)
        
        print(f"💾 已存档：{filename}")
    
    def cmd_load(self, _):
        """读档"""
        filename = input("存档名（默认save_勇者.json）：").strip()
        if not filename:
            filename = f"save_{self.player.name}.json"
        if not filename.endswith('.json'):
            filename += '.json'
        
        try:
            with open(filename, 'r', encoding='utf-8') as f:
                data = json.load(f)
            
            p = data['player']
            self.player = Player(**p)
            self.turn_count = data['turn']
            print(f"📂 读取成功！欢迎回来，{self.player.name}！")
        except FileNotFoundError:
            print("找不到存档文件")
        except Exception as e:
            print(f"读档失败：{e}")
    
    def cmd_talk(self, _):
        """对话"""
        if self.player.location == "城堡":
            print("\n👑 国王说：")
            print("   '勇者啊，请救救我的女儿！龙王在西北方的城堡...'")
            print("   '带上光之武器才能伤害它！'")
        elif self.player.location == "道具店":
            print("\n🧙 店主说：")
            print("   '药草是冒险者的生命线，多带几个吧！'")
        else:
            print("\n🗣️ 附近没有人可以交谈...")
    
    def cmd_rest(self, _):
        """休息恢复（仅限安全区）"""
        loc = self.world.LOCATIONS[self.player.location]
        if loc['danger_level'] == 0:
            self.player.hp = self.player.max_hp
            self.player.mp = self.player.max_mp
            print("💤 你休息了一会儿，HP和MP完全恢复了！")
        else:
            print("❌ 这里太危险了，无法休息！")
    
    def cmd_shop(self, _):
        """打开商店"""
        loc = self.world.LOCATIONS[self.player.location]
        if loc.get('shop'):
            self.state = "shop"
        else:
            print("这里不是商店...")
    
    def cmd_ai_info(self, _):
        """AI说明"""
        print("""
🤖 关于AI叙事功能：

当前状态：""" + ("✅ 已启用" if isinstance(self.narrator, DeepSeekNarrator) and self.narrator.enabled else "❌ 未启用") + """

如何开启：
1. 访问 https://platform.deepseek.com/ 注册账号
2. 创建API Key（免费额度够用很久）
3. 设置环境变量：
   Windows: set DEEPSEEK_API_KEY=你的密钥
   Mac/Linux: export DEEPSEEK_API_KEY=你的密钥
4. 重新启动游戏

AI模式会为你生成独特的场景描述，每次探索都不同！
        """)
    
    def cmd_help(self, _):
        """帮助"""
        print("""
📖 指令列表：

基础：
  移动 [地点] / m [地点]  - 移动到其他地方
  探索 / e                - 调查当前地点
  状态 / s                - 查看勇者状态
  物品 / i                - 查看背包
  魔法 / m                - 查看魔法
  休息                    - 在安全区恢复HP/MP
  商店                    - 打开商店（在商店地点）

互动：
  talk                    - 与NPC对话
  
系统：
  存档                    - 保存进度
  读取                    - 读取进度
  ai                      - AI功能说明
  帮助 / h                - 显示此帮助
  退出 / quit             - 结束游戏

战斗时：
  攻击 / a                - 普通攻击
  魔法 / m                - 使用魔法
  物品 / i                - 使用道具
  逃跑 / r                - 尝试逃跑
        """)
    
    def cmd_quit(self, _):
        """退出"""
        print("确定要退出吗？未存档进度将丢失")
        if input("(yes/no): ").lower() in ['yes', 'y', '是']:
            self.running = False
    
    def game_over(self):
        """游戏结束"""
        print("""
    ╔══════════════════════════════════════╗
    ║           G A M E   O V E R          ║
    ║              游 戏 结 束              ║
    ╠══════════════════════════════════════╣
    ║     勇者倒下了，但传说永不终结...      ║
    ╚══════════════════════════════════════╝
        """)
        print(f"⏱️  存活回合：{self.turn_count}")
        print(f"⭐ 最终等级：Lv.{self.player.level}")
        print(f"💰 持有金币：{self.player.gold}G")

def main():
    game = GameEngine()
    game.start()
    print("\n感谢游玩！再会，勇者！")

if __name__ == "__main__":
    main()
