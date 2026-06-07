#!/usr/bin/env python
"""
同花顺 iFinD MCP Bridge Server
通过 iFinD HTTP API 为 Claude Code 提供金融数据查询能力
"""

import os
import json
import time
import logging
from typing import Any

import requests
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# ── 配置 ──────────────────────────────────────────────
REFRESH_TOKEN = os.environ.get("IFIND_REFRESH_TOKEN", "")
BASE_URL = "https://quantapi.51ifind.com/api/v1"
DEFAULT_TIMEOUT = 30

# 内部状态：access_token 缓存
_access_token: str = ""
_access_token_expiry: float = 0.0

logging.basicConfig(level=logging.INFO, format="%(asctime)s [MCP-iFinD] %(message)s")
log = logging.getLogger("ifind-mcp")

# ── Token 管理 ─────────────────────────────────────────

def get_access_token() -> str:
    """获取或刷新 access_token，自动缓存"""
    global _access_token, _access_token_expiry

    if _access_token and time.time() < _access_token_expiry - 60:
        return _access_token

    if not REFRESH_TOKEN:
        raise RuntimeError("IFIND_REFRESH_TOKEN 环境变量未设置")

    log.info("正在获取 access_token...")
    resp = requests.post(
        f"{BASE_URL}/get_access_token",
        headers={
            "Content-Type": "application/json",
            "refresh_token": REFRESH_TOKEN,
        },
        timeout=DEFAULT_TIMEOUT,
    )
    data = resp.json()
    if data.get("errorcode") != 0 and data.get("errcode") != 0:
        raise RuntimeError(f"获取 access_token 失败: {data.get('errmsg', resp.text)}")

    _access_token = data["data"]["access_token"]
    # 解析过期时间，提前 5 分钟刷新
    expiry_str = data["data"]["expired_time"]  # "2026-06-14 11:17:00"
    from datetime import datetime
    expiry_dt = datetime.strptime(expiry_str, "%Y-%m-%d %H:%M:%S")
    _access_token_expiry = expiry_dt.timestamp()
    log.info(f"access_token 已刷新，过期时间: {expiry_str}")
    return _access_token


def api_post(endpoint: str, body: dict) -> dict:
    """调用 iFinD HTTP API"""
    token = get_access_token()
    resp = requests.post(
        f"{BASE_URL}/{endpoint}",
        json=body,
        headers={
            "Content-Type": "application/json",
            "access_token": token,
        },
        timeout=DEFAULT_TIMEOUT,
    )
    resp.raise_for_status()
    return resp.json()


# ── MCP Server ─────────────────────────────────────────

server = Server("ifind-mcp")


@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="ifind_token_status",
            description="查看当前 access_token 状态和有效期",
            inputSchema={
                "type": "object",
                "properties": {},
                "required": [],
            },
        ),
        Tool(
            name="ifind_real_time_quotation",
            description="获取实时行情数据。支持股票代码列表和指标列表。",
            inputSchema={
                "type": "object",
                "properties": {
                    "codes": {
                        "type": "string",
                        "description": "股票代码，逗号分隔，格式如 300033.SZ,600030.SH",
                    },
                    "indicators": {
                        "type": "string",
                        "description": "指标字段，逗号分隔，如 open,high,low,latest,volume,amount,changeRatio",
                    },
                },
                "required": ["codes", "indicators"],
            },
        ),
        Tool(
            name="ifind_history_quotation",
            description="获取历史K线数据。支持日/周/月/季/年线。",
            inputSchema={
                "type": "object",
                "properties": {
                    "codes": {
                        "type": "string",
                        "description": "股票代码，逗号分隔",
                    },
                    "indicators": {
                        "type": "string",
                        "description": "行情指标，逗号分隔，如 open,high,low,close,volume,amount",
                    },
                    "period": {
                        "type": "string",
                        "description": "K线周期: D(日), W(周), M(月), Q(季), Y(年)",
                        "default": "D",
                    },
                    "start_date": {
                        "type": "string",
                        "description": "开始日期 YYYY-MM-DD 或 YYYYMMDD",
                    },
                    "end_date": {
                        "type": "string",
                        "description": "结束日期 YYYY-MM-DD 或 YYYYMMDD",
                    },
                    "fq_type": {
                        "type": "string",
                        "description": "复权类型: 前复权(默认), 后复权, 不复权",
                        "default": "前复权",
                    },
                },
                "required": ["codes", "indicators", "start_date", "end_date"],
            },
        ),
        Tool(
            name="ifind_basic_data",
            description="获取基础数据（财务指标、估值指标等），如 ROE、PE、市值等。",
            inputSchema={
                "type": "object",
                "properties": {
                    "codes": {
                        "type": "string",
                        "description": "股票代码，逗号分隔",
                    },
                    "indicators": {
                        "type": "string",
                        "description": "指标代码，逗号分隔。常用: ths_roe_stock(ROE), ths_pe_ttm_stock(PE-TTM), ths_total_equity_atoopc_stock(归母净资产)",
                    },
                    "report_date": {
                        "type": "string",
                        "description": "报告期，如 20241231 表示2024年报",
                    },
                },
                "required": ["codes", "indicators"],
            },
        ),
        Tool(
            name="ifind_date_sequence",
            description="获取日期序列数据（时间序列），支持指定频率和填充规则。适合获取一段时间内的财务数据变化。",
            inputSchema={
                "type": "object",
                "properties": {
                    "codes": {
                        "type": "string",
                        "description": "股票代码，逗号分隔",
                    },
                    "indicators": {
                        "type": "string",
                        "description": "指标代码，逗号分隔",
                    },
                    "start_date": {
                        "type": "string",
                        "description": "开始日期 YYYYMMDD",
                    },
                    "end_date": {
                        "type": "string",
                        "description": "结束日期 YYYYMMDD",
                    },
                    "days": {
                        "type": "string",
                        "description": "日类型: Tradedays(交易日), Alldays(日历日)",
                        "default": "Tradedays",
                    },
                    "interval": {
                        "type": "string",
                        "description": "频率: D(日), W(周), M(月), Q(季), S(半年), Y(年)",
                        "default": "M",
                    },
                    "fill": {
                        "type": "string",
                        "description": "填充规则: Previous(前值填充), Blank(空值)",
                        "default": "Previous",
                    },
                },
                "required": ["codes", "indicators", "start_date", "end_date"],
            },
        ),
        Tool(
            name="ifind_data_pool",
            description="获取数据池信息，如板块成分股列表、指数成分股等。",
            inputSchema={
                "type": "object",
                "properties": {
                    "pool_type": {
                        "type": "string",
                        "description": "数据池类型，如 block(板块), index(指数)",
                        "default": "block",
                    },
                    "pool_id": {
                        "type": "string",
                        "description": "数据池ID，如 001005260(沪深300). 不填则返回所有板块列表.",
                    },
                    "date": {
                        "type": "string",
                        "description": "查询日期 YYYYMMDD",
                    },
                    "fields": {
                        "type": "string",
                        "description": "返回字段，逗号分隔。常用: security_name,thscode,date",
                        "default": "date:Y,security_name:Y,thscode:Y",
                    },
                },
                "required": [],
            },
        ),
        Tool(
            name="ifind_search_indicator",
            description="搜索可用的财务指标。输入关键词返回匹配的指标代码和说明，用于其他接口的 indicators 参数。",
            inputSchema={
                "type": "object",
                "properties": {
                    "keyword": {
                        "type": "string",
                        "description": "搜索关键词，如 ROE, PE, 净利润, 营业收入",
                    },
                },
                "required": ["keyword"],
            },
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    try:
        if name == "ifind_token_status":
            token = get_access_token() if REFRESH_TOKEN else ""
            masked = token[:10] + "..." + token[-10:] if token else "(无)"
            return [TextContent(
                type="text",
                text=json.dumps({
                    "status": "有效" if token else "未配置",
                    "access_token": masked,
                    "refresh_token_expiry": "2026-06-26 15:13:59",
                }, ensure_ascii=False, indent=2),
            )]

        elif name == "ifind_real_time_quotation":
            codes = arguments["codes"]
            indicators = arguments["indicators"]
            result = api_post("real_time_quotation", {
                "codes": codes,
                "indicators": indicators,
            })
            return [TextContent(type="text", text=json.dumps(result, ensure_ascii=False, indent=2))]

        elif name == "ifind_history_quotation":
            codes = arguments["codes"]
            indicators = arguments["indicators"]
            start = arguments["start_date"]
            end = arguments["end_date"]
            period = arguments.get("period", "D")
            fq = arguments.get("fq_type", "前复权")

            # 构造 functionpara
            func_para = f"period:{period},pricetype:1,rptcategory:0,fqdate:1900-01-01,hb:YSHB,fill:Previous"
            result = api_post("history_quotation", {
                "codes": codes,
                "indicators": indicators,
                "functionpara": func_para,
                "startdate": start.replace("-", ""),
                "enddate": end.replace("-", ""),
            })
            return [TextContent(type="text", text=json.dumps(result, ensure_ascii=False, indent=2))]

        elif name == "ifind_basic_data":
            codes = arguments["codes"]
            indicators = arguments["indicators"]
            report_date = arguments.get("report_date", "")

            indipara = []
            for ind in indicators.split(","):
                indipara.append({
                    "indicator": ind.strip(),
                    "indiparams": [report_date] if report_date else [""],
                })

            result = api_post("basic_data_service", {
                "codes": codes,
                "indipara": indipara,
            })
            return [TextContent(type="text", text=json.dumps(result, ensure_ascii=False, indent=2))]

        elif name == "ifind_date_sequence":
            codes = arguments["codes"]
            indicators = arguments["indicators"]
            start = arguments["start_date"]
            end = arguments["end_date"]
            days = arguments.get("days", "Tradedays")
            interval = arguments.get("interval", "M")
            fill = arguments.get("fill", "Previous")

            func_para = f"Days:{days},Fill:{fill},Interval:{interval}"
            indipara = []
            for ind in indicators.split(","):
                indipara.append({
                    "indicator": ind.strip(),
                    "indiparams": ["", "100"],
                })

            result = api_post("date_sequence", {
                "codes": codes,
                "startdate": start,
                "enddate": end,
                "functionpara": func_para,
                "indipara": indipara,
            })
            return [TextContent(type="text", text=json.dumps(result, ensure_ascii=False, indent=2))]

        elif name == "ifind_data_pool":
            pool_type = arguments.get("pool_type", "block")
            pool_id = arguments.get("pool_id", "")
            date = arguments.get("date", "")
            fields = arguments.get("fields", "date:Y,security_name:Y,thscode:Y")

            result = api_post("data_pool", {
                "pooltype": pool_type,
                "poolid": pool_id if pool_id else "",
                "date": date,
                "field": fields,
            })
            return [TextContent(type="text", text=json.dumps(result, ensure_ascii=False, indent=2))]

        elif name == "ifind_search_indicator":
            keyword = arguments["keyword"]
            # iFinD 没有直接的搜索 API，这里提供一个常用指标速查
            common_indicators = {
                "ROE": "ths_roe_stock — 净资产收益率(ROE)",
                "ROA": "ths_roa_stock — 总资产收益率(ROA)",
                "PE": "ths_pe_ttm_stock — 市盈率PE(TTM)",
                "PB": "ths_pb_lf_stock — 市净率PB(LF)",
                "市值": "ths_total_market_value_stock — 总市值",
                "净利润": "ths_net_profit_atoopc_stock — 归母净利润",
                "营业收入": "ths_operating_revenue_stock — 营业总收入",
                "毛利率": "ths_gross_profit_margin_stock — 销售毛利率",
                "净利率": "ths_net_profit_margin_stock — 销售净利率",
                "资产负债率": "ths_debt_to_asset_ratio_stock — 资产负债率",
                "EPS": "ths_eps_stock — 每股收益EPS",
                "每股净资产": "ths_bps_stock — 每股净资产BPS",
                "现金流": "ths_cash_flow_ratio_stock — 现金流量比率",
                "股息率": "ths_dividend_yield_stock — 股息率",
                "换手率": "ths_turnover_rate_stock — 换手率",
                "涨跌幅": "changeRatio — 涨跌幅",
                "成交量": "volume — 成交量",
                "成交额": "amount — 成交额",
            }

            matches = {}
            kw = keyword.upper()
            for k, v in common_indicators.items():
                if kw in k.upper() or kw in v.upper():
                    matches[k] = v

            return [TextContent(
                type="text",
                text=json.dumps(
                    {"keyword": keyword, "matches": matches if matches else common_indicators},
                    ensure_ascii=False, indent=2,
                ),
            )]

        else:
            return [TextContent(type="text", text=f"未知工具: {name}")]

    except Exception as e:
        log.exception(f"工具 {name} 执行失败")
        return [TextContent(type="text", text=json.dumps({"error": str(e)}, ensure_ascii=False))]


# ── 入口 ───────────────────────────────────────────────

async def main():
    log.info("iFinD MCP Server 启动中...")
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream, server.create_initialization_options())


if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
