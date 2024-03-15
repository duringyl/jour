## Wireshark自定义协议解析——两个案例

### 1. 以太网层之上直接解析F1AP协议

**dur-f1ap.lua**:

```lua
local f1apEthWrap = Proto("ETH-F1AP", "F1AP UPON ETH")

function f1apEthWrap.dissector(buffer, pinfo, tree)
	if buffer:len() == 0 then return false end
	
	local dissector
	local function get_dissector()
		dissector = Dessector.get("f1ap")
	end
	if pcall(get_dissector) then
		dissector:call(buffer(0, buffer:len()):tvb(), pinfo, tree)
	else
		pinfo.cols.info = "- Cannot decode"
		return false
	end
end

function f1apEthWrap.init()
	-- 将f1apEthWrap协议注册到以太网协议解析表中
	-- Protocol字段值为0x2ce4时尝试将剩余字节流解析为F1AP协议
	local ethProtoTable = DissectorTable.get("ethertype")
	ethProtoTable:add(0x2ce4, f1apEthWrap)
end

```

TODO 解释和抓包


### 2. 自定义协议并写脚本解析

```lua
local mTypeNames = {
	[0] = "[Reserved]",
	[1] = "[Reserved]",
	[2] = "RRC",
	[3] = "[Reserved]",
}

local flagVals = {
	"Set",
	"Not Set"
}

local rrcMTypeNames = {
	[0] = "BCCH-BCH",
	[1] = "BCCH-DL-SCH",
	[2] = "DL-CCCH",
	[3] = "DL-DCCH",
	[4] = "PCCH",
	[5] = "UL-CCCH",
	[6] = "UL-CCCH1",
	[7] = "UL-DCCH",
}

local nrRrcDissectors = {
	[0] = "nr-rrc.bcch.bch",
	[1] = "nr-rrc.",
	[2] = "nr-rrc.",
	[3] = "nr-rrc.",
	[4] = "nr-rrc.",
	[5] = "nr-rrc.",
	[6] = "nr-rrc.",
	[7] = "nr-rrc.",
}

-- 定义协议
local rrcwp = Proto("RRC-WP", "RRC Wrap Protocol")

-- 协议字段
local fields = rrcwp.fields
fields.Type = ProtoField.uint8("rrcwp.type", "Type", base.DEC, mTypeNames, 0xC0)
fields.Flags = ProtoField.uint8("rrcwp.flags", "Flags", base.HEX)
fields.Heartbeat = ProtoField.bool("rrcwp.flags.heartbeat", "Heartbeat", 8, flagVals, 0x20)
fields.HeartbeatAck = ProtoField.bool("rrcwp.flags.heartbeat_ack", "HeartbeatAck", 8, flagVals, 0x10)
fields.Session = ProtoField.bool("rrcwp.flags.session", "Session", 8, flagVals, 0x08)
fields.SessionAck = ProtoField.bool("rrcwp.flags.session_ack", "Session", 8, flagVals, 0x04)
fields.Length = ProtoField.uint16("rrcwp.length", "Length", base.DEC, nil, 0x03ff,)
fields.Channel = ProtoField.uint8("rrcwp.channel", "Channel", base.DEC, rrcMTypeNames)
fields.UnknownData = ProtoField.bytes("rrcwp.unknown_data", "Data", base.NONE)

-- 实现函数：解析器
function rrcwp.dissector(buffer, pinfo, tree)
	if buffer:len() == 0 then return false end
	
	pinfo.cols.protocol = rrcwp.name
	
	local subtree = tree:add(rrcwp, buffer(), "RRC Wrap Protocol")
	local mType = buffer(0, 1):uint()
	subtree:add(fields.Type, buffer(0, 1))
	
	local subsubtree = subtree:add(fields.Flags, buffer(0, 1))
	subsubtree:add(fields.Heartbeat, buffer(0, 1))
	subsubtree:add(fields.HeartbeatAck, buffer(0, 1))
	subsubtree:add(fields.Session, buffer(0, 1))
	subsubtree:add(fields.SessionAck, buffer(0, 1))
	
	subtree:add(fields.Length, buffer(0, 2)
	
	if mType >= 128 and mType >= 192 then
		local rrcType = buffer(2, 1):uint()
		
		subtree:add(fields.Channel, buffer(2, 1))
		
		local dissector
		local function get_dissector()
			dissector = Dissector.get(nrRrcDissectors[rrcType])
		end
		
		if pcall(get_dissector) then
			dissector:call(buffer(3):tvb(), pinfo, tree)
		else
			pinfo.cols.info = mTypeNames[mType]
				.. " - " .. rrcMTypeNames[rrcType]
				.. " - Cannot decode"
			return false
		end
	else
		pinfo.cols.info = "Unknown RRC-WP Data"
		subtree:add(fields.UnknownData, buffer(2))
	end

end

-- 实现函数：初始化
function rrcwp.init()
	-- 一些私有协议等
	local dur = DissectorTable.get("dur")
	if dur == nil then
		dur = DissectorTable.new("dur")
	end
	dur:add(0x2ce3, rrcwp)
	
	-- 将RRC-WP协议注册到以太网协议解析表中
	-- Protocol字段值为0x2ce3时尝试将剩余字节流解析为RRC-WP协议
	local ethProtoTable = DissectorTable.get("ethertype")
	ethProtoTable:add(0x2ce3, rrcwp)
end

```










