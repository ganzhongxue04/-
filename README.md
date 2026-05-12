# -
模型自主查询本月的法定节假日
"""
react思路：
当用户问："这个月有几个法定节假日？"
系统会：
    思考：需要先知道当前月份
    行动：调用get_current_date获取当前日期
    观察：得到"2025年11月15日"
    思考：现在查询9月的节假日
    行动：调用search_holidays("11月")
    观察：得到"2025年11月没有法定节假日"
    最终回答：给出完整答案
"""
from openai import OpenAI
from datetime import datetime
# 定义函数访问模型
from openai import OpenAI

client = OpenAI(base_url="https://dashscope.aliyuncs.com/compatible-mode/v1")
def call_qwen(prompt):
    response = client.chat.completions.create(
        model="qwen-plus",
        messages=[
            {"role": "system", "content": prompt}
        ]
    )
    return response.choices[0].message.content
# 定义工具函数
def get_current_date():
    return f"{datetime.today().month}月"

def search_holidays(month):
    # 模拟节假日数据
    holidays = {
        "1月": ["元旦：1月1日"],
        "2月": ["春节：1月28日-2月3日"],
        "3月": [],
        "4月": ["清明节：4月4日-6日"],
        "5月": ["劳动节：5月1日-5日", "端午节：5月31日-6月2日"],
        "6月": [],
        "7月": [],
        "8月": [],
        "9月": [],  # 2025年9月没有法定节假日
        "10月": ["中秋节：10月6日-8日", "国庆节：10月1日-7日"],
        "11月": [],
        "12月": ["元旦：12月31日"]
    }
    holidays_list = holidays.get(month,[])
    if holidays_list:
        return f"2025年{month}有以下法定节假日：\n" + "\n".join(holidays_list)
    else:
        return f"2025年{month}没有法定节假日。"
TOOLS = {
    "get_current_date": get_current_date,
    "search_holidays": search_holidays
}
def parse_response(response:str):
    # 要解析的字符串是
    """
    Thought: 我需要先获取当前日期，以确定“这个月”是哪个月，然后查询该月的法定节假日数量。
    Action: get_current_date
    Action Input:
    """
    thought=""
    action=''
    action_input = ""
    list_line = response.strip().split("\n")
    for line in list_line:
        line = line.strip() # todo 去除空格
        if line.startswith("Thought:"):
            thought = line.replace("Thought:","").strip()
        elif line.startswith("Action:"):
            action = line.replace("Action:","").strip()
        elif line.startswith("Action Input:"):
            action_input = line.replace("Action Input:","").strip()

    return thought,action,action_input




# react动作
def react(question):
    # 1.定义react的动作最多6次
    max_steps = 6
    # 2.定义变量保存没次思考行动观察的上下文.
    steps = []
    for i in range(max_steps):
        # 3.拼接上下文
        context = "\n".join(steps)
        # 4.拼接prompt
        prompt = f"""
                你是一个使用ReAct范式的智能代理，必须严格按以下格式输出：
                一个字都不能错，不能多加内容，必须按照以下格式输出：
                Thought: <你的思考>
                Action: <要执行的动作，从 [get_current_date, search_holidays] 中选择，当知道结果后返回: Action:Final Answer>
                Action Input: <如果Action是动作名称，则填写该工具的输入参数；如果Action是Final Answer，则填写最终答案>
                当前上下文：
                {context}
                问题：{question}
                【工具说明】
                - get_current_date
                  作用：获取月份
                  传入参数：无
                  返回结果：月份，如10月
                - search_holidays
                  作用：查询节假日
                  传入参数：月份，如2月
                  返回结果：节假日信息，如 1月3日~1月9日 春节
                  
                【正确示例】
                - 示例1（调用工具）：
                  Thought: 要回答本月有哪些法定假日，首先需要获取当前月份，才能针对性查询假期。
                  Action: get_current_date
                  Action Input: ''
                - 示例2（输出最终答案）：
                  Thought: 已通过查询得知当前月份是2024年10月，该月的法定假日是10月1日-7日（国庆节），可以给出最终答案。
                  Action: Final Answer
                  Action Input: 2024年10月的法定节假日是国庆节，放假时间为10月1日至7日
                """
        # 5.调用模型
        print(f"第{i+1}次询问模型的提示词：{prompt}\n\n")
        response = call_qwen(prompt)
        print(f"模型的回答:\n{response}\n\n")
        # 模型的回答:
        # Thought: 我需要先获取当前日期，以确定“这个月”是哪个月，然后查询该月的法定节假日数量。
        # Action: get_current_date
        # Action Input:

        # 6.解析模型回答
        thought, action, action_input = parse_response(response)

        # 7 检查解析结果 是否正确，比如action，thought是否有值。
        if not thought and  not action:
            print("模型回答有误，请检查模型是否正确")
            steps.append(f"ERROR:模型回答有误，内容是{response}")
            continue

        # 8 检查action 是否有正确结果
        if action == "Final Answer":
            print(f"最终回答：{action_input}")
            return action_input

        # 9.检查记录模型的回答。当做下次的上下文。
        steps.append(f"Thought: {thought}")
        steps.append(f"Action: {action}")

        # 10 运行对应的工具函数 -- 观察阶段。
        if action in TOOLS:
            print(f"执行工具函数{action},参数是{action_input}")
            if action == "get_current_date":
                toos_data = TOOLS[action]() # todo 调用工具里面的函数
            else:
                toos_data = TOOLS[action](action_input)
            print(f'执行工具函数{action}的返回结果是{toos_data}\n\n')
            steps.append(f"Observation: {toos_data}")
        else:
            if action != "":
                steps.append(f"ERROR: 没有找到对应的工具函数{action}")

    print("没有找到对应的答案")
    return "没有找到对应的答案"




if __name__ == '__main__':
    # print(get_current_date())
    # print(search_holidays(get_current_date()))
    # print(search_holidays('12月'))

    # s1 = """
    # Thought: 我需要先获取当前日期，以确定“这个月”是哪个月，然后查询该月的法定节假日数量。
    # Action: get_current_date
    # Action Input:
    # """
    # thought, action, action_input = parse_response(s1)
    # print(thought)
    # print(action)
    # print(action_input)


    react("这个月有哪些法定节假日？")
