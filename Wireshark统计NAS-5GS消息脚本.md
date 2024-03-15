### wireshark用于统计NAS-5GS消息数量的lua脚本(*wireshark自定义菜单*)

##### nas_5gs_statistics.lua:

```lua
--[[
    2023/04/15 - 2023/04/15
    nas_5gs_statistics.lua v1.0
    Wireshark Lua Script for Statistics of NAS-5GS

    by duringyl

    usage: 1. load this script in wireshark; 2. select menu: 统计/Statistics -> NAS-5GS
]]--

local nas_mm_field = Field.new("nas_5gs.mm.message_type")
local nas_sm_field = Field.new("nas_5gs.sm.message_type")

-- nas mm消息显示修正补丁: 有的Registration request消息包含在Security mode complete内, 统计值需要修正
local function nas_mm_patch(nas)
    local regist = nas["Registration request (0x41)"] or 0
    local smc_compl = nas["Security mode complete (0x5e)"] or 0

    nas["[Registration request] Not In [Security mode complete]"] = regist - smc_compl

    --[[local ul_nas = nas["UL NAS transport (0x67)"]
    if ul_nas ~= nil then
        nas["UL NAS transport (0x67)"] = tostring(ul_nas) .. " incredible"
    ]]--
end


local function nas_5gs_statistics()
    local filter_mm = "ngap && nas_5gs.mm.message_type"
    local filter_sm = "ngap && nas_5gs.sm.message_type"

    local tw = TextWindow.new("Statistics Of NAS-5GS")

    local tap = Listener.new(nil, filter_mm)
    local tap2 = Listener.new(nil, filter_sm)

    tw:set_atclose(function()
        tap:remove()
        tap2:remove()
    end)


    local nas_mm = {}
    local cnt_mm = 0

    function tap.packet(pinfo, tvb, tapinfo)
        local nas_5gs = { nas_mm_field() }
        for _, msg in ipairs(nas_5gs) do
            local name = msg.display or "unexpected"    -- display, name
            local num = nas_mm[name] or 0
            nas_mm[name] = num + 1
            cnt_mm = cnt_mm + 1
        end
    end

    function tap.draw(t)
        tw:append("\nfilter mm: " .. filter_mm .. "\n")
        tw:append("mm total: " .. cnt_mm .. "\n")
        nas_mm_patch(nas_mm)
        for nas_name, num in pairs(nas_mm) do
            tw:append(num .. "\t" .. nas_name .. "\n")
        end
    end

    function tap.reset()
        tw:clear()
        nas_mm = {}
        nas_sm = {}
    end


    local nas_sm = {}
    local cnt_sm = 0

    function tap2.packet(pinfo, tvb, tapinfo)
        local nas_5gs = { nas_sm_field() }
        for _, msg in ipairs(nas_5gs) do
            local name = msg.display or "unexpected"    -- display, name
            local num = nas_sm[name] or 0
            nas_sm[name] = num + 1
            cnt_sm = cnt_sm + 1
        end
    end

    function tap2.draw(t)
        tw:append("\nfilter sm: " .. filter_sm .. "\n")
        tw:append("sm total: " .. cnt_sm .. "\n")
        for nas_name, num in pairs(nas_sm) do
            tw:append(num .. "\t" .. nas_name .. "\n")
        end
    end

    function tap2.reset()
        tw:clear()
        nas_sm = {}
    end

    retap_packets()

end

register_menu("NAS-5GS", nas_5gs_statistics, MENU_STAT_UNSORTED)    -- Test/Packets


-- other backup↓
--[[
usage:
    1. Tools -> Lua -> Console 打开控制台;
    2. Tools -> Lua -> Evaluate打开命令输入窗口;
    3. 把下面print_all_things函数的代码复制到Evaluate; 提交后在Console查看打印输出
]]--
--[[
local function print_all_things()
    for _, d in ipairs(Dissector.list()) do
        print(d)
    end

    for _, d in ipairs(DissectorTable.list()) do
        print(d)
    end

    -- 需要非常之久
    for _, f in ipairs(Field.list()) do
        print(f)
    end
end

]]--
```
