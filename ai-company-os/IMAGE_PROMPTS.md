# 背景图生成提示词

参考图：office_bg.png（山屋像素场景，冬季雪地风格，俯视45度视角）

生成要求：
- 分辨率：1280 × 720 像素
- 风格：与参考图完全一致的像素艺术风格（pixel art），俯视45度等距视角
- 色深：16-32色限制，保持像素艺术质感
- 不需要任何文字、UI元素、角色

---

## F1_BG — 公共协作层

**参考图使用方式：** 参考图中的木屋内部区域（壁炉、沙发、咖啡机）作为整体风格基准，地板纹理参考石板地面部分。

**提示词（英文，给 Nano Banana）：**

```
Pixel art office interior background, 1280x720, isometric top-down 45-degree view, same art style as the reference image (mountain lodge cozy pixel art).

The scene is divided into 4 natural zones with no hard walls between them, only implied by furniture and floor texture changes:

Top-left zone (MEETING ROOM): Large wooden conference table with 6 chairs around it, a whiteboard on the wall with pixel art scribbles, warm ceiling lamp, wooden floor tiles similar to the reference lodge floor.

Top-right zone (ARCHIVE ROOM): Floor-to-ceiling wooden bookshelves filled with files and books (similar to the bookshelf in the reference image), a reading lamp, filing cabinets, stone tile floor.

Bottom-left zone (RESIDENT AGENT WORKSTATION): 3-4 wooden desks with retro computer monitors (similar to the computer desk in reference), desk lamps, potted plants between desks, stone tile floor matching the reference.

Bottom-right zone (REST AREA): Comfortable sofa and armchair (reuse the sofa/chair style from reference), small coffee table, floor rug, warm fireplace glow (similar to reference fireplace), sleeping cat on a cushion.

Overall atmosphere: cozy warm indoor office, winter light from windows, consistent pixel art fidelity with the reference image. No characters, no UI, no text labels.
```

---

## B1_BG — 基础设施层

**参考图使用方式：** 参考图右上角的服务器机柜区域作为风格基准，扩展为整个场景。

**提示词（英文，给 Nano Banana）：**

```
Pixel art server room / data center background, 1280x720, isometric top-down 45-degree view, same pixel art style as the reference image.

The scene is divided into 4 natural zones:

Left zone (SERVER ROOM): Multiple tall server racks with blinking green/red LED lights (expand the server cabinet style from the reference image), cooling pipes on the ceiling, concrete floor with subtle grid lines, warning light on one rack.

Center-right zone (DATABASE ROOM): Rows of shorter storage units/NAS devices, blue indicator lights, cable management trays on the floor, slightly darker atmosphere than server room.

Top-right zone (MCP TOOL RACK): Open shelving units with various electronic equipment, routers, switches, labeled tool boxes, workbench with soldering equipment.

Bottom (KNOWLEDGE CORRIDOR): Wide open corridor connecting all rooms, floor lined with glowing data cables in the ground, subtle blue ambient lighting, a few terminal monitors on pedestals.

Overall atmosphere: cool blue-tinted underground tech room, fluorescent lighting with some blue accent lights, industrial feel, consistent pixel art style with the reference image. No characters, no UI, no text.
```

---

## PROJECT_BG — 项目工作层

**参考图使用方式：** 整体参考图的石板地面区域 + 书架区域，色调偏向中性工作环境。

**提示词（英文，给 Nano Banana）：**

```
Pixel art open-plan office workspace background, 1280x720, isometric top-down 45-degree view, same pixel art style as the reference image.

The scene is an open workspace without room dividers, organized into functional areas:

Upper area (TEAM LEAD WORKSTATIONS): 4 evenly spaced wooden desks arranged in a 2x2 grid, each desk has a retro monitor, keyboard, desk lamp, and a small name plate holder (empty). Desks are separated by small potted plants and low shelves. Warm wooden floor.

Lower area (WORKER ZONE): Slightly smaller desks in a 4-wide row, more compact, some desks have dual monitors, scattered coffee cups and sticky notes, slightly different floor texture (lighter wood).

Left wall: Large window showing exterior city view (pixel art city skyline, night with lights), giving ambient light to the room.

Right wall: Whiteboard / corkboard with pixel art sticky notes and diagrams pinned on it.

Center: Open walkway between team lead and worker zones, wide enough for characters to walk through.

Overall atmosphere: modern creative office, neutral warm lighting, productive environment, consistent pixel art fidelity with the reference image. No characters, no UI, no text labels.
```

---

## 使用说明

1. 将以上三段英文提示词分别发给 Nano Banana
2. 每次发送时同时附上 `office_bg.png` 作为参考图（style reference）
3. 生成后检查以下几点：
   - 视角是否与参考图一致（俯视45度等距）
   - 像素风格是否匹配（不能出现平滑渐变或写实风格）
   - 尺寸是否为 1280×720
   - 四个区域是否自然划分、边界清晰
4. 如果风格偏差较大，在提示词前加：`In the exact same pixel art style as the attached reference image, with identical perspective angle and color palette:`
