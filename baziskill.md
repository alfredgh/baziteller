import datetime
import math
from typing import Dict, List, Tuple, Optional

class BaziCalculator:
    """
    八字测算核心类
    支持公历日期输入，自动计算四柱八字、十神、五行、纳音、大运、流年等
    基于节气划分月柱和年柱，日柱采用公式计算，时柱按五鼠遁，大运依据阳年顺逆排法
    """
    
    # 天干地支基础数据
    GAN = ["甲", "乙", "丙", "丁", "戊", "己", "庚", "辛", "壬", "癸"]
    ZHI = ["子", "丑", "寅", "卯", "辰", "巳", "午", "未", "申", "酉", "戌", "亥"]
    
    # 五行映射
    GAN_WU_XING = {
        "甲": "木", "乙": "木", "丙": "火", "丁": "火", "戊": "土", "己": "土",
        "庚": "金", "辛": "金", "壬": "水", "癸": "水"
    }
    ZHI_WU_XING = {
        "寅": "木", "卯": "木", "辰": "土", "巳": "火", "午": "火", "未": "土",
        "申": "金", "酉": "金", "戌": "土", "亥": "水", "子": "水", "丑": "土"
    }
    
    # 地支藏干（主气）
    ZHI_CANG_GAN = {
        "子": ["癸"], "丑": ["己", "癸", "辛"], "寅": ["甲", "丙", "戊"], "卯": ["乙"],
        "辰": ["戊", "乙", "癸"], "巳": ["丙", "庚", "戊"], "午": ["丁", "己"], "未": ["己", "丁", "乙"],
        "申": ["庚", "壬", "戊"], "酉": ["辛"], "戌": ["戊", "辛", "丁"], "亥": ["壬", "甲"]
    }
    
    # 十神关系（日干为主）
    SHI_SHEN_MAP = {
        ("同","同"): "比肩", ("同","异"): "劫财",
        ("生","同"): "食神", ("生","异"): "伤官",
        ("被生","同"): "偏印", ("被生","异"): "正印",
        ("克","同"): "偏财", ("克","异"): "正财",
        ("被克","同"): "七杀", ("被克","异"): "正官"
    }
    
    # 节气表（月支分界点）: 格式 (节气名称, 月支, 对应公历日期函数)
    # 为简化且保证准确性，内置2020-2030年主要节气日期（立春、惊蛰、清明、立夏、芒种、小暑、立秋、白露、寒露、立冬、大雪、小寒）
    # 实际使用时可扩充至更广年份，此处提供核心年份数据，支持1900-2100可通过算法扩展，但为演示提供常用范围
    SOLAR_TERMS = {
        2020: {"立春": (2,4), "惊蛰": (3,5), "清明": (4,4), "立夏": (5,5), "芒种": (6,5), "小暑": (7,6),
               "立秋": (8,7), "白露": (9,7), "寒露": (10,8), "立冬": (11,7), "大雪": (12,7), "小寒": (1,6)},
        2021: {"立春": (2,3), "惊蛰": (3,5), "清明": (4,4), "立夏": (5,5), "芒种": (6,5), "小暑": (7,7),
               "立秋": (8,7), "白露": (9,7), "寒露": (10,8), "立冬": (11,7), "大雪": (12,7), "小寒": (1,5)},
        2022: {"立春": (2,4), "惊蛰": (3,5), "清明": (4,5), "立夏": (5,5), "芒种": (6,6), "小暑": (7,7),
               "立秋": (8,7), "白露": (9,7), "寒露": (10,8), "立冬": (11,7), "大雪": (12,7), "小寒": (1,5)},
        2023: {"立春": (2,4), "惊蛰": (3,6), "清明": (4,5), "立夏": (5,6), "芒种": (6,6), "小暑": (7,7),
               "立秋": (8,8), "白露": (9,8), "寒露": (10,8), "立冬": (11,8), "大雪": (12,7), "小寒": (1,5)},
        2024: {"立春": (2,4), "惊蛰": (3,5), "清明": (4,4), "立夏": (5,5), "芒种": (6,5), "小暑": (7,6),
               "立秋": (8,7), "白露": (9,7), "寒露": (10,8), "立冬": (11,7), "大雪": (12,6), "小寒": (1,6)},
        2025: {"立春": (2,3), "惊蛰": (3,5), "清明": (4,4), "立夏": (5,5), "芒种": (6,5), "小暑": (7,7),
               "立秋": (8,7), "白露": (9,7), "寒露": (10,8), "立冬": (11,7), "大雪": (12,7), "小寒": (1,5)},
        2026: {"立春": (2,4), "惊蛰": (3,5), "清明": (4,5), "立夏": (5,5), "芒种": (6,5), "小暑": (7,7),
               "立秋": (8,7), "白露": (9,7), "寒露": (10,8), "立冬": (11,7), "大雪": (12,7), "小寒": (1,5)},
        2027: {"立春": (2,4), "惊蛰": (3,6), "清明": (4,5), "立夏": (5,6), "芒种": (6,6), "小暑": (7,7),
               "立秋": (8,8), "白露": (9,8), "寒露": (10,9), "立冬": (11,8), "大雪": (12,7), "小寒": (1,6)},
        2028: {"立春": (2,5), "惊蛰": (3,5), "清明": (4,4), "立夏": (5,5), "芒种": (6,5), "小暑": (7,6),
               "立秋": (8,7), "白露": (9,7), "寒露": (10,8), "立冬": (11,7), "大雪": (12,6), "小寒": (1,6)},
        2029: {"立春": (2,4), "惊蛰": (3,5), "清明": (4,4), "立夏": (5,5), "芒种": (6,5), "小暑": (7,7),
               "立秋": (8,7), "白露": (9,7), "寒露": (10,8), "立冬": (11,7), "大雪": (12,7), "小寒": (1,5)},
        2030: {"立春": (2,4), "惊蛰": (3,5), "清明": (4,5), "立夏": (5,5), "芒种": (6,5), "小暑": (7,7),
               "立秋": (8,7), "白露": (9,7), "寒露": (10,8), "立冬": (11,7), "大雪": (12,7), "小寒": (1,5)},
    }
    # 补充部分年份（2000-2019）的立春数据用于年柱计算
    ADDITIONAL_LICHUN = {
        2000: (2,4), 2001: (2,4), 2002: (2,4), 2003: (2,4), 2004: (2,4), 2005: (2,4),
        2006: (2,4), 2007: (2,4), 2008: (2,4), 2009: (2,4), 2010: (2,4), 2011: (2,4),
        2012: (2,4), 2013: (2,4), 2014: (2,4), 2015: (2,4), 2016: (2,4), 2017: (2,3),
        2018: (2,4), 2019: (2,4)
    }
    
    def __init__(self, year: int, month: int, day: int, hour: int, minute: int, gender: str = "男"):
        """
        初始化八字计算
        :param year: 公历年份
        :param month: 月份 1-12
        :param day: 日期
        :param hour: 小时（0-23），时柱以23点为界，23-0点为子时
        :param minute: 分钟
        :param gender: 性别 "男" 或 "女"
        """
        self.birth_date = datetime.date(year, month, day)
        self.year = year
        self.month = month
        self.day = day
        self.hour = hour
        self.minute = minute
        self.gender = gender
        
        # 处理时柱子时边界（23:00-0:59为当日或次日子时，八字以子时为起始，23-0点属于当日的子时）
        self.zhi_index = (hour + 1) // 2 % 12  # 0-11对应子丑寅卯...
        if hour == 23:
            self.zhi_index = 0  # 子时
        
        self.year_gan = ""
        self.year_zhi = ""
        self.month_gan = ""
        self.month_zhi = ""
        self.day_gan = ""
        self.day_zhi = ""
        self.hour_gan = ""
        self.hour_zhi = self.ZHI[self.zhi_index]
        
        # 存储四柱
        self.bazi = {}
        self.nayin = {}
        self.ten_gods = {}
        
    def _get_lichun_date(self, y: int) -> Tuple[int, int]:
        """获取指定年份的立春日期（月，日）"""
        if y in self.ADDITIONAL_LICHUN:
            return self.ADDITIONAL_LICHUN[y]
        if y in self.SOLAR_TERMS:
            return self.SOLAR_TERMS[y]["立春"]
        # 默认近似2月4日
        return (2, 4)
    
    def _get_solar_term_date(self, y: int, term_name: str) -> Optional[Tuple[int, int]]:
        """获取指定年份的节气日期"""
        if y in self.SOLAR_TERMS and term_name in self.SOLAR_TERMS[y]:
            return self.SOLAR_TERMS[y][term_name]
        return None
    
    def compute_year_column(self):
        """计算年柱（以立春为界）"""
        lichun_month, lichun_day = self._get_lichun_date(self.year)
        birth = self.birth_date
        lichun = datetime.date(self.year, lichun_month, lichun_day)
        if birth < lichun:
            # 属于前一年
            year_for_ganzhi = self.year - 1
        else:
            year_for_ganzhi = self.year
        # 年干支基数: 甲子为1，公式 (year-4) % 60
        offset = (year_for_ganzhi - 4) % 60
        self.year_gan = self.GAN[offset % 10]
        self.year_zhi = self.ZHI[offset % 12]
        return self.year_gan, self.year_zhi
    
    def compute_month_column(self):
        """计算月柱（根据节气分界，月支固定，月干用五虎遁）"""
        # 确定月支：根据节气划分，每月对应一个节
        # 定义节气-月支映射（按顺序）
        term_month_map = [
            ("立春", "寅"), ("惊蛰", "卯"), ("清明", "辰"), ("立夏", "巳"),
            ("芒种", "午"), ("小暑", "未"), ("立秋", "申"), ("白露", "酉"),
            ("寒露", "戌"), ("立冬", "亥"), ("大雪", "子"), ("小寒", "丑")
        ]
        # 查找出生日期所在的节气区间
        year_for_term = self.year
        # 如果生日在立春之前，月份干支属于前一年的月柱？但月份计算仍以当前年节气为准，但年柱已调整，月份按实际节气区间
        # 先获取本年所有节气日期，如果生日在第一个节气（立春）之前，则属于前一年的最后一个月（丑月）
        lichun = self._get_lichun_date(self.year)
        birth = self.birth_date
        lichun_date = datetime.date(self.year, lichun[0], lichun[1])
        if birth < lichun_date:
            # 属于前一年的丑月，需用前一年的年干来遁月干
            year_for_gan = self.year - 1
            month_zhi = "丑"
            # 月干根据年干用五虎遁
            year_gan_for_month = self.GAN[(year_for_gan - 4) % 10]
            month_gan = self._wuhudun(year_gan_for_month, month_zhi)
            self.month_zhi = month_zhi
            self.month_gan = month_gan
            return month_gan, month_zhi
        
        # 正常情况：遍历节气确定月支
        # 先构造当年的所有节气日期列表
        terms = []
        for term_name, zhi in term_month_map:
            date_tuple = self._get_solar_term_date(self.year, term_name)
            if date_tuple:
                terms.append((datetime.date(self.year, date_tuple[0], date_tuple[1]), zhi, term_name))
        # 按日期排序
        terms.sort(key=lambda x: x[0])
        month_zhi = None
        # 找到出生日期所在的区间
        for i in range(len(terms)):
            term_date = terms[i][0]
            if birth < term_date:
                if i == 0:
                    # 不可能，因为已经处理过立春前
                    month_zhi = "丑"
                else:
                    month_zhi = terms[i-1][1]
                break
        if month_zhi is None:
            month_zhi = terms[-1][1]  # 最后一个节气之后
        
        # 月干用五虎遁，基于年干
        year_gan_index = (self.year - 4) % 10
        year_gan = self.GAN[year_gan_index]
        month_gan = self._wuhudun(year_gan, month_zhi)
        self.month_zhi = month_zhi
        self.month_gan = month_gan
        return month_gan, month_zhi
    
    def _wuhudun(self, year_gan: str, month_zhi: str) -> str:
        """五虎遁求月干: 甲己之年丙作首，乙庚之岁戊为头，丙辛必定寻庚起，丁壬壬位顺行流，若问戊癸何方发，甲寅之上好追求"""
        start_map = {
            "甲": "丙", "己": "丙",
            "乙": "戊", "庚": "戊",
            "丙": "庚", "辛": "庚",
            "丁": "壬", "壬": "壬",
            "戊": "甲", "癸": "甲"
        }
        start_gan = start_map[year_gan]
        zhi_order = self.ZHI
        start_index = zhi_order.index("寅")
        target_index = zhi_order.index(month_zhi)
        offset = (target_index - start_index) % 12
        gan_index = self.GAN.index(start_gan)
        target_gan_index = (gan_index + offset) % 10
        return self.GAN[target_gan_index]
    
    def compute_day_column(self):
        """计算日柱（公历日期，使用公式）"""
        # 公式：日干支基数 = (年尾二位数+7)*5 +15 + (年尾二位数+19)/4 ，取整，然后mod60
        # 再计算该日期是该年的第几天，基数+天数 mod60
        year = self.year
        month = self.month
        day = self.day
        # 处理1月和2月视为前一年的13月和14月
        if month == 1 or month == 2:
            year -= 1
            month += 12
        century = year // 100
        year_last_two = year % 100
        # 日干支基数计算
        base = (century // 4) * 5 + 15 + (century % 4) * 5 + 30  # 简化的蔡勒公式变种
        base = (year_last_two + 7) * 5 + 15 + (year_last_two + 19) // 4
        # 计算该日期是该年的第几天
        is_leap = (year % 4 == 0 and year % 100 != 0) or (year % 400 == 0)
        days_in_month = [31, 29 if is_leap else 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31]
        day_of_year = sum(days_in_month[:month-1]) + day
        total = base + day_of_year
        idx = total % 60
        self.day_gan = self.GAN[idx % 10]
        self.day_zhi = self.ZHI[idx % 12]
        return self.day_gan, self.day_zhi
    
    def compute_hour_column(self):
        """计算时柱（五鼠遁）"""
        # 根据日干确定时干起始（甲己日甲子时，乙庚日丙子，丙辛日戊子，丁壬日庚子，戊癸日壬子）
        start_map = {
            "甲": "甲", "己": "甲",
            "乙": "丙", "庚": "丙",
            "丙": "戊", "辛": "戊",
            "丁": "庚", "壬": "庚",
            "戊": "壬", "癸": "壬"
        }
        start_gan = start_map[self.day_gan]
        zhi_list = self.ZHI
        start_zhi = "子"
        start_index = zhi_list.index(start_zhi)
        target_index = zhi_list.index(self.hour_zhi)
        offset = (target_index - start_index) % 12
        gan_index = self.GAN.index(start_gan)
        target_gan_index = (gan_index + offset) % 10
        self.hour_gan = self.GAN[target_gan_index]
        return self.hour_gan, self.hour_zhi
    
    def get_nayin(self, ganzhi: str) -> str:
        """获取纳音五行（简化版，仅年柱纳音）"""
        nayin_dict = {
            "甲子": "海中金", "乙丑": "海中金", "丙寅": "炉中火", "丁卯": "炉中火",
            "戊辰": "大林木", "己巳": "大林木", "庚午": "路旁土", "辛未": "路旁土",
            "壬申": "剑锋金", "癸酉": "剑锋金", "甲戌": "山头火", "乙亥": "山头火",
            "丙子": "涧下水", "丁丑": "涧下水", "戊寅": "城头土", "己卯": "城头土",
            "庚辰": "白蜡金", "辛巳": "白蜡金", "壬午": "杨柳木", "癸未": "杨柳木",
            "甲申": "泉中水", "乙酉": "泉中水", "丙戌": "屋上土", "丁亥": "屋上土",
            "戊子": "霹雳火", "己丑": "霹雳火", "庚寅": "松柏木", "辛卯": "松柏木",
            "壬辰": "长流水", "癸巳": "长流水", "甲午": "沙中金", "乙未": "沙中金",
            "丙申": "山下火", "丁酉": "山下火", "戊戌": "平地木", "己亥": "平地木",
            "庚子": "壁上土", "辛丑": "壁上土", "壬寅": "金箔金", "癸卯": "金箔金",
            "甲辰": "覆灯火", "乙巳": "覆灯火", "丙午": "天河水", "丁未": "天河水",
            "戊申": "大驿土", "己酉": "大驿土", "庚戌": "钗钏金", "辛亥": "钗钏金",
            "壬子": "桑柘木", "癸丑": "桑柘木", "甲寅": "大溪水", "乙卯": "大溪水",
            "丙辰": "沙中土", "丁巳": "沙中土", "戊午": "天上火", "己未": "天上火",
            "庚申": "石榴木", "辛酉": "石榴木", "壬戌": "大海水", "癸亥": "大海水"
        }
        return nayin_dict.get(ganzhi, "未知")
    
    def get_ten_god(self, gan: str, ri_gan: str) -> str:
        """根据日干求十神"""
        # 五行生克关系：同阴阳为同，异阴阳为异
        gan_yin_yang = {g: i%2 for i,g in enumerate(self.GAN)}  # 0阳1阴
        ri_yin_yang = gan_yin_yang[ri_gan]
        tar_yin_yang = gan_yin_yang[gan]
        relation = ""
        wuxing_ri = self.GAN_WU_XING[ri_gan]
        wuxing_tar = self.GAN_WU_XING[gan]
        # 生克关系
        # 五行生克: 木火土金水相生，木土水金火相克（具体）
        sheng = {"木":"火","火":"土","土":"金","金":"水","水":"木"}
        ke = {"木":"土","土":"水","水":"火","火":"金","金":"木"}
        if wuxing_tar == wuxing_ri:
            relation = "同"
        elif sheng[wuxing_ri] == wuxing_tar:
            relation = "生"
        elif sheng[wuxing_tar] == wuxing_ri:
            relation = "被生"
        elif ke[wuxing_ri] == wuxing_tar:
            relation = "克"
        elif ke[wuxing_tar] == wuxing_ri:
            relation = "被克"
        yinyang_key = "同" if ri_yin_yang == tar_yin_yang else "异"
        return self.SHI_SHEN_MAP.get((relation, yinyang_key), "未知")
    
    def compute_ten_gods(self):
        """计算四柱天干十神及地支主气十神"""
        ri_gan = self.day_gan
        pillars = {
            "年柱": (self.year_gan, self.year_zhi),
            "月柱": (self.month_gan, self.month_zhi),
            "日柱": (self.day_gan, self.day_zhi),
            "时柱": (self.hour_gan, self.hour_zhi)
        }
        result = {}
        for pillar_name, (gan, zhi) in pillars.items():
            gan_god = self.get_ten_god(gan, ri_gan) if pillar_name != "日柱" else "日元"
            # 地支主气十神
            main_gan = self.ZHI_CANG_GAN[zhi][0]
            zhi_god = self.get_ten_god(main_gan, ri_gan)
            result[pillar_name] = {"天干十神": gan_god, "地支主气十神": zhi_god}
        return result
    
    def get_wuxing_of_bazi(self):
        """获取四柱天干地支的五行"""
        pillars = {
            "年柱": (self.year_gan, self.year_zhi),
            "月柱": (self.month_gan, self.month_zhi),
            "日柱": (self.day_gan, self.day_zhi),
            "时柱": (self.hour_gan, self.hour_zhi)
        }
        wuxing = {}
        for name, (gan, zhi) in pillars.items():
            wuxing[name] = {"天干五行": self.GAN_WU_XING[gan], "地支五行": self.ZHI_WU_XING[zhi]}
        return wuxing
    
    def compute_da_yun(self):
        """排大运（阳年男顺排，阴年女顺排，反之逆排）"""
        # 阳年: 甲丙戊庚壬
        yang_gan = ["甲", "丙", "戊", "庚", "壬"]
        year_gan = self.year_gan
        is_yang = year_gan in yang_gan
        # 顺逆: 阳男阴女顺排，阴男阳女逆排
        if (is_yang and self.gender == "男") or (not is_yang and self.gender == "女"):
            direction = "顺"
        else:
            direction = "逆"
        # 起运岁数计算：顺排找下一个节气，逆排找上一个节气
        # 为简化示例，此处返回排运方向和示例大运干支（实际完整计算需逐月推算）
        # 返回月柱顺逆排列的前十个大运
        month_ganzhi = self.month_gan + self.month_zhi
        gan_index = self.GAN.index(self.month_gan)
        zhi_index = self.ZHI.index(self.month_zhi)
        da_yun_list = []
        for i in range(1, 11):
            if direction == "顺":
                new_gan = self.GAN[(gan_index + i) % 10]
                new_zhi = self.ZHI[(zhi_index + i) % 12]
            else:
                new_gan = self.GAN[(gan_index - i) % 10]
                new_zhi = self.ZHI[(zhi_index - i) % 12]
            da_yun_list.append(new_gan + new_zhi)
        return {"direction": direction, "da_yun_10_years": da_yun_list}
    
    def get_current_liu_nian(self, year: int) -> str:
        """获取指定年份的流年干支"""
        offset = (year - 4) % 60
        return self.GAN[offset % 10] + self.ZHI[offset % 12]
    
    def analyze(self, target_year: Optional[int] = None) -> Dict:
        """综合分析，返回完整八字信息"""
        self.compute_year_column()
        self.compute_month_column()
        self.compute_day_column()
        self.compute_hour_column()
        
        bazi = {
            "年柱": self.year_gan + self.year_zhi,
            "月柱": self.month_gan + self.month_zhi,
            "日柱": self.day_gan + self.day_zhi,
            "时柱": self.hour_gan + self.hour_zhi
        }
        nayin = {
            "年柱纳音": self.get_nayin(bazi["年柱"]),
        }
        ten_gods = self.compute_ten_gods()
        wuxing = self.get_wuxing_of_bazi()
        da_yun = self.compute_da_yun()
        liu_nian = self.get_current_liu_nian(target_year or datetime.datetime.now().year)
        
        return {
            "出生时间": f"{self.year}年{self.month}月{self.day}日 {self.hour}:{self.minute}",
            "性别": self.gender,
            "四柱八字": bazi,
            "纳音": nayin,
            "五行": wuxing,
            "十神": ten_gods,
            "大运": da_yun,
            "当前流年": liu_nian
        }


# Skill 调用示例
def bazi_skill(birth_date: str, birth_time: str, gender: str) -> dict:
    """
    八字测算Skill入口
    :param birth_date: 公历日期 "YYYY-MM-DD"
    :param birth_time: 时间 "HH:MM" 24小时制
    :param gender: "男" 或 "女"
    :return: 八字分析结果字典
    """
    try:
        date_parts = birth_date.split("-")
        year, month, day = int(date_parts[0]), int(date_parts[1]), int(date_parts[2])
        time_parts = birth_time.split(":")
        hour, minute = int(time_parts[0]), int(time_parts[1])
        calculator = BaziCalculator(year, month, day, hour, minute, gender)
        result = calculator.analyze()
        return result
    except Exception as e:
        return {"error": f"计算失败: {str(e)}"}

# 使用说明
if __name__ == "__main__":
    # 测试示例：1990年5月15日 14:30 出生 男
    res = bazi_skill("1990-05-15", "14:30", "男")
    import pprint
    pprint.pprint(res)
