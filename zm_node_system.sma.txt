/*=========================================================================
[ExTasY] A* Node Pathfinding system
-Started: 2022.09.14
===========================================================================*/
#include <amxmodx>
#include <amxmisc>
#include <fakemeta>
#include <fakemeta_util>
#include <xs>

//Node létrehozásához, debughoz kapcsold be.
#define DEVMODE

//Csak akkor vidd feljebb, ha ennél több node is előfordul az egyik pályán.
#define MAX_NODES 256

/*===========================================================================/
/--------------------------------Definitions---------------------------------/
/============================================================================*/
new g_dot, g_File[128], g_bhg
new Float:g_iPoints[2][3], g_iNode, g_iActionType
new g_Nodes

enum _:Items{
	Float:CORNER_0[3],
	Float:CORNER_1[3],
	Float:CORNER_2[3],
	Float:CORNER_3[3],
	Float:CORNER_4[3]
}
new Array:g_aNodes[4];

enum {
	BIT_POINT_SET1,
	BIT_POINT_SET2,
	BIT_TRACK_EXISTING_POINTS,
	BIT_NEW_POINT_SET1,
	BIT_NEW_POINT_SET2
}
new g_bits;

enum _:Actions{
	NORMAL, //Piros
	DUCK, //Sárga
	JUMP, //Lila
	LADDER, //Rózsaszín
	SPECIAL //Világoskék
}
new const g_cActionColors[Actions][3] = {
	{255, 0, 0},
	{235, 239, 0},
	{116, 0, 239},
	{239, 0, 232},
	{0, 239, 217}
}

#define SetBit(%0,%1) (%0 |= (1 << %1))
#define ClearBit(%0,%1) (%0 &= ~(1 << %1))
#define GetBit(%0,%1) (%0 & (1 << %1))

#define MAIN 0
#define CONNECTED 1
#define ACTION 2
#define MIDPOINT 3

#define MAX_CONNECTABLE_NODES 4

/*===========================================================================/
/------------------------------Global Functions------------------------------/
/============================================================================*/
public plugin_init(){
	register_plugin("[ZM] - Node System", "0.0.1", "DexoN")

	#if defined DEVMODE
		register_clcmd("ActionType", "@ActionType")
		register_clcmd("say /nodes", "Menu_Node")
		register_concmd("path", "@ManualPath")
		set_task(0.7, "review", .flags = "b")
	#endif
}

public plugin_cfg(){
	g_Nodes = 0

	get_configsdir(g_File, charsmax(g_File))
	formatex(g_File, charsmax(g_File), "%s/NodeSystem", g_File)

	if(!dir_exists(g_File))
		mkdir(g_File)

	new sMap[32]; get_mapname(sMap, charsmax(sMap))

	formatex(g_File, charsmax(g_File), "%s/%s.cfg", g_File, sMap)

	new f = fopen(g_File, "r")
	if(!f){
		log_amx("Nem található node fájl ezen a pályán.")
		return;
	}

	new Line[128], szO[7][8]

	while(!feof(f)){
		fgets(f, Line, charsmax(Line))

		if(strlen(Line) < 2)
			break;

		parse(Line, szO[0], 7, szO[1], 7, szO[2], 7, szO[3], 7, szO[4], 7, szO[5], 7, szO[6], 7)

		for(new i; i < 3; i++) g_iPoints[0][i] = str_to_float(szO[i])
		for(new i; i < 3; i++) g_iPoints[1][i] = str_to_float(szO[i+3])
		g_iActionType = str_to_num(szO[6])

		InsertNode(g_iNode)
		SetNodeAction(g_iNode)
		SetNodeMidPoint(g_iNode)
	}
	fclose(f)

	log_amx("%d node sikeresen betöltve.", g_Nodes)

	if(g_Nodes > 1)
		CreateNetwork()
}

public plugin_precache() {
	g_dot = precache_model("sprites/dot.spr");

	g_aNodes[MAIN] = ArrayCreate(Items)
	g_aNodes[CONNECTED] = ArrayCreate(Items)
	g_aNodes[ACTION] = ArrayCreate()
	g_aNodes[MIDPOINT] = ArrayCreate(3)
}

/*===========================================================================/
/---------------------------------Natives------------------------------------/
/============================================================================*/
public plugin_natives() {
	register_native("zm_npc_find_path", "_npc_find_path", 1);
	register_native("zm_get_user_node", "_get_user_node", 1)
	register_native("zm_is_ent_in_node", "_is_ent_in_node", 1)
	register_native("zm_get_random_node", "_get_random_node", 1)
	register_native("zm_get_node_action", "_get_node_action", 1)
	register_native("zm_get_user_closest_node", "_get_user_closest_node", 1)
	register_native("zm_get_node_closest_point", "_get_node_closest_point", 1)
}

public Array:_npc_find_path(npc, player)
	return FindPathToNode(npc, player);

public _get_user_node(iId)
	return GetNodeByOrigin(iId);

public _is_ent_in_node(iEnt, Node){
	static Float:fO[3], l_Array[Items]
	pev(iEnt, pev_origin, fO)
	ArrayGetArray(g_aNodes[MAIN], Node, l_Array)

	if((IsPosInRectangle(fO, l_Array[CORNER_0], l_Array[CORNER_1]))
	||(IsPosInRectangle(fO, l_Array[CORNER_0], l_Array[CORNER_1]))
	||(IsPosInRectangle(fO, l_Array[CORNER_0], l_Array[CORNER_1]))
	||(IsPosInRectangle(fO, l_Array[CORNER_0], l_Array[CORNER_1]))
	||(IsPosInRectangle(fO, l_Array[CORNER_0], l_Array[CORNER_1]))) return 1

	return 0
}

public _get_random_node(iEnt, Type){
	new iNode = GetNodeByOrigin(iEnt)
	new rNode
	for(new i; i < g_Nodes*MAX_CONNECTABLE_NODES; i++){
		rNode = random(g_Nodes)

		if(rNode == iNode || ArrayGetCell(g_aNodes[ACTION], rNode) != Type)
			continue;

		return rNode;
	}
	return 0
}

public _get_node_action(iNode)
	return ArrayGetCell(g_aNodes[ACTION], iNode);

public _get_user_closest_node(iEnt){
	static Float:fDist, l_Array[Items], Float:Mid[3], Float:fO[3], l_Node;
	fDist = 5000.0
	pev(iEnt, pev_origin, fO)

	for(new i; i < g_Nodes; i++){
		ArrayGetArray(g_aNodes[MAIN], i, l_Array)

		GetMidpoint(l_Array[CORNER_0], l_Array[CORNER_1], Mid)

		if(xs_vec_distance(Mid, fO) < fDist){
			fDist = xs_vec_distance(Mid, fO)
			l_Node = i;
		}
	}
	return l_Node
}

public Array:_get_node_closest_point(iEnt, Node){
	static Array:aGoodPoint, Float:Point[3], Float:fO[3]
	aGoodPoint = ArrayCreate(3)

	pev(iEnt, pev_origin, fO)

	GetClosestPointToOrigin(fO, Node, Point)

	ArrayPushArray(aGoodPoint, Point)

	return aGoodPoint;
}

/*===========================================================================/
/----------------------------------Menus-------------------------------------/
/============================================================================*/
public Menu_Node(iId){
	new menu = menu_create(fmt("Node menu^n\yNodes: \r%d db", g_Nodes), "@MenuH_Node")

	menu_additem(menu, "Node létrehozása")
	menu_additem(menu, "Node törlése")
	menu_additem(menu, "Node szerkesztése")
	menu_additem(menu, fmt("Közeli nodeok mutatása %s", g_bhg ? "\r[\yON\r]" : "\r[\dOFF\r]"))
	menu_additem(menu, "Node csatlakozási pontjai")
	menu_additem(menu, "Nodeok mentése")

	menu_display(iId, menu)
}

@MenuH_Node(iId, menu, item){
	if(item==MENU_EXIT)
		return PLUGIN_HANDLED;

	switch(item){
		case 0: Menu_Create_Node(iId)
		case 1: Menu_Delete_Node(iId)
		case 2: Modify_Node(iId)
		case 3: {
			g_bhg = g_bhg ? 0 : iId
			Menu_Node(iId)
		}
		case 4: GetNodeNetwork(GetNodeByOrigin(iId))
		case 5: SaveNodes()
	}

	return PLUGIN_HANDLED;
}

public Menu_Create_Node(iId){
	new sLine[128], iLen;
	iLen += formatex(sLine[iLen], charsmax(sLine), GetBit(g_bits, BIT_POINT_SET1) ? fmt("^n\yPoint 1 - \r[\w%.2f %.2f %.2f\r]", g_iPoints[0][0], g_iPoints[0][1], g_iPoints[0][2]) : "")
	iLen += formatex(sLine[iLen], charsmax(sLine)-iLen, GetBit(g_bits, BIT_POINT_SET2) ? fmt("^n\yPoint 2 - \r[\w%.2f %.2f %.2f\r]", g_iPoints[1][0], g_iPoints[1][1], g_iPoints[1][2]) : "")
	iLen += formatex(sLine[iLen], charsmax(sLine)-iLen, g_iNode ? fmt("^n\yNode %d", g_iNode-1) : "")

	new menu = menu_create(fmt("Node létrehozás menü%s", sLine), "@MenuH_Create_Node")

	menu_additem(menu, "Pont beállítása.")
	menu_additem(menu, "Nézett pont felkapása.")
	menu_additem(menu, fmt("%sJelöld meg a nodeomat.", (GetBit(g_bits, BIT_POINT_SET1) && GetBit(g_bits, BIT_POINT_SET2)) ? "\w" : "\d"))
	menu_additem(menu, fmt("Cselekvéstípus: \r[\d%d\r]", g_iActionType))
	menu_additem(menu, "Node létrehozása.")

	menu_display(iId, menu)
}

@MenuH_Create_Node(iId, menu, item){
	menu_destroy(menu)

	if(item == MENU_EXIT){
		return PLUGIN_HANDLED;
	}

	switch(item){
		case 0: {
			fm_get_aim_origin(iId, (!GetBit(g_bits, BIT_POINT_SET2) || GetBit(g_bits, BIT_NEW_POINT_SET2)) ? g_iPoints[1] : g_iPoints[0])
			if(GetBit(g_bits, BIT_POINT_SET1)){
				SetBit(g_bits, BIT_POINT_SET2)
			}else SetBit(g_bits, BIT_POINT_SET1);
			if(GetBit(g_bits, BIT_NEW_POINT_SET1))
				ClearBit(g_bits, BIT_NEW_POINT_SET1)
			else if(GetBit(g_bits, BIT_NEW_POINT_SET2))
				ClearBit(g_bits, BIT_NEW_POINT_SET2);
			SetBit(g_bits, BIT_TRACK_EXISTING_POINTS)
			if(!task_exists(iId))
				set_task(0.1, "ShowBeam", iId, .flags = "b")
		}
		case 1: PickUpViewingPoint(iId)
		case 2: (GetBit(g_bits, BIT_POINT_SET1) && GetBit(g_bits, BIT_POINT_SET2)) ? DrawRectangle(g_iPoints[0], g_iPoints[1], 50, g_cActionColors[g_iActionType][0], g_cActionColors[g_iActionType][1], g_cActionColors[g_iActionType][2]) : client_print_color(0,0, "Nincs 2 pontod.")
		case 3: {
			client_cmd(iId, "messagemode ActionType")
			return PLUGIN_HANDLED;
		}
		case 4: {
			new Float:f[3]; pev(iId, pev_origin, f)
			InsertNode(g_iNode)
			SetNodeAction(g_iNode)
			SetNodeMidPoint(g_iNode)
			ClearBits()
			CreateNetwork()
			Menu_Node(iId)
			return PLUGIN_HANDLED;
		}
	}

	Menu_Create_Node(iId)
	
	return PLUGIN_HANDLED;
}

public Menu_Delete_Node(iId){
	new iNode, l_Array[Items]
	if((iNode = GetNodeByAim(iId)) == -1){
		Menu_Node(iId)
		return;
	}

	ArrayGetArray(g_aNodes[MAIN], iNode, l_Array)

	xs_vec_copy(l_Array[CORNER_0], g_iPoints[0])
	xs_vec_copy(l_Array[CORNER_1], g_iPoints[1])

	SetBit(g_bits, BIT_POINT_SET2);
	SetBit(g_bits, BIT_POINT_SET1);

	g_iNode = iNode+1;

	new sLine[128], iLen;
	iLen += formatex(sLine[iLen], charsmax(sLine), GetBit(g_bits, BIT_POINT_SET1) ? fmt("^n\yPoint 1 - \r[\w%.2f %.2f %.2f\r]", g_iPoints[0][0], g_iPoints[0][1], g_iPoints[0][2]) : "")
	iLen += formatex(sLine[iLen], charsmax(sLine)-iLen, GetBit(g_bits, BIT_POINT_SET2) ? fmt("^n\yPoint 2 - \r[\w%.2f %.2f %.2f\r]", g_iPoints[1][0], g_iPoints[1][1], g_iPoints[1][2]) : "")
	iLen += formatex(sLine[iLen], charsmax(sLine)-iLen, g_iNode ? fmt("^n\yNode %d", g_iNode) : "")

	new menu = menu_create(fmt("Biztosan törlöd ezt a node-ot?%s", sLine), "@MenuH_Delete_Node")

	menu_additem(menu, "Igen.")
	menu_additem(menu, "Nem.")

	menu_display(iId, menu)
}

@MenuH_Delete_Node(iId, menu, item){
	menu_destroy(menu)

	if(item == MENU_EXIT){
		return PLUGIN_HANDLED;
	}

	switch(item){
		case 0: {
			DeleteNode(g_iNode-1)
			ClearBits()
			CreateNetwork()
			Menu_Node(iId)
		}
		case 1: {
			ClearBits()
			Menu_Node(iId)
		}
	}
	
	return PLUGIN_HANDLED;
}

/*===========================================================================/
/----------------------------Internal Functions------------------------------/
/============================================================================*/
GetClosestPointToOrigin(Float:fO[3], Node, Float:GoodPoint[3]){
	static l_Array[Items], Float:fDiff, Float:Lowest;
	ArrayGetArray(g_aNodes[MAIN], Node, l_Array)

	fDiff = 5000.0

	Lowest = l_Array[CORNER_0][2] > l_Array[CORNER_1][2] ? l_Array[CORNER_1][2] : l_Array[CORNER_0][2]

	if(xs_vec_distance(fO, l_Array[CORNER_0]) < fDiff){
		fDiff = xs_vec_distance(fO, l_Array[CORNER_0])
		xs_vec_copy(l_Array[CORNER_0], GoodPoint)
	}
	else if(xs_vec_distance(fO, l_Array[CORNER_1]) < fDiff){
		fDiff = xs_vec_distance(fO, l_Array[CORNER_1])
		xs_vec_copy(l_Array[CORNER_1], GoodPoint)
	}
	else if(xs_vec_distance(fO, l_Array[CORNER_2]) < fDiff){
		fDiff = xs_vec_distance(fO, l_Array[CORNER_2])
		xs_vec_copy(l_Array[CORNER_2], GoodPoint)
	}
	else if(xs_vec_distance(fO, l_Array[CORNER_3]) < fDiff){
		fDiff = xs_vec_distance(fO, l_Array[CORNER_3])
		xs_vec_copy(l_Array[CORNER_3], GoodPoint)
	}
	else if(xs_vec_distance(fO, l_Array[CORNER_4]) < fDiff){
		fDiff = xs_vec_distance(fO, l_Array[CORNER_4])
		xs_vec_copy(l_Array[CORNER_4], GoodPoint)
	}

	GoodPoint[2] = Lowest
}

#if defined DEVMODE
	public review(){
		if(!g_bhg)
			return;

		static node; node = GetNodeByOrigin(g_bhg)
		if(node != -1)
			GetNodeNetwork(node)
	}

	@ManualPath(iId){
	new sString[16]
	read_args(sString, charsmax(sString))

	remove_quotes(sString)

	new sItems[2][8]
	parse(sString, sItems[0], 7, sItems[1], 7)

	FindPathToNode(str_to_num(sItems[0]), str_to_num(sItems[1]))
}
#endif

SaveNodes(){
	if(!g_Nodes){
		client_print_color(0,0, "Nincs egy darab node sem.")
		return;
	}

	HighlightNode(-2)

	new l_Array[Items], l_Type;
	new f = fopen(g_File, "w")
	if(f){
		for(new i; i < g_Nodes; i++){
			ArrayGetArray(g_aNodes[MAIN], i, l_Array)
			l_Type = ArrayGetCell(g_aNodes[ACTION], i)
			fputs(f, fmt("%.2f %.2f %.2f %.2f %.2f %.2f %d^n",\
			l_Array[CORNER_0][0], l_Array[CORNER_0][1], l_Array[CORNER_0][2], l_Array[CORNER_1][0], l_Array[CORNER_1][1], l_Array[CORNER_1][2], l_Type))
		}
		fclose(f)
	}
	client_print_color(0,0, "^4Koordináták elmentve. ^3:)")
}

@ActionType(iId){
	new sString[4]
	read_args(sString, charsmax(sString))

	remove_quotes(sString)

	g_iActionType = str_to_num(sString)
	Menu_Create_Node(iId)
}

DeleteNode(Node){
	ArrayDeleteItem(g_aNodes[MAIN], Node)
	ArrayDeleteItem(g_aNodes[CONNECTED], Node)
	ArrayDeleteItem(g_aNodes[ACTION], Node)
	ArrayDeleteItem(g_aNodes[MIDPOINT], Node)
	g_Nodes--
}

InsertNode(Node = 0){
	new Float:fO[Items]
	xs_vec_copy(g_iPoints[0], fO[CORNER_0]);
	xs_vec_copy(g_iPoints[1], fO[CORNER_1]);
	
	new Float:Lowest = g_iPoints[0][2] < g_iPoints[1][2] ? g_iPoints[0][2] : g_iPoints[1][2]
	new Float:fHighest[3]; xs_vec_copy(g_iPoints[0][2] > g_iPoints[1][2] ? g_iPoints[0] : g_iPoints[1], fHighest)
	fHighest[2] = Lowest; xs_vec_copy(fHighest, fO[CORNER_4])

	xs_vec_copy(g_iPoints[0], fO[CORNER_2]); fO[CORNER_2][2] = Lowest; fO[CORNER_2][0] = g_iPoints[1][0];
	xs_vec_copy(g_iPoints[1], fO[CORNER_3]); fO[CORNER_3][2] = Lowest; fO[CORNER_3][0] = g_iPoints[0][0];

	if(!Node){
		ArrayPushArray(g_aNodes[MAIN], fO)
		g_Nodes++
	}
	else
		ArraySetArray(g_aNodes[MAIN], g_iNode-1, fO)
}

SetNodeAction(Node = 0){
	if(!Node)
		ArrayPushCell(g_aNodes[ACTION], g_iActionType)
	else
		ArraySetCell(g_aNodes[ACTION], g_iNode-1, g_iActionType)
}

SetNodeMidPoint(Node = 0){
	new Float:Mid[3]
	GetMidpoint(g_iPoints[0], g_iPoints[1], Mid)
	if(!Node)
		ArrayPushArray(g_aNodes[MIDPOINT], Mid)
	else
		ArraySetArray(g_aNodes[MIDPOINT], g_iNode-1, Mid)
}

CreateNetwork(){
	new l_Array[Items], l_Array2[Items], l_Array3[MAX_CONNECTABLE_NODES]
	new aSize = g_Nodes
	client_print_color(0,0, "^3Node létrehozva: ^4#^1%d", g_Nodes)
	if(aSize == 1 || !aSize){
		client_print_color(0,0, "return")
		return;
	}
	
	ArrayClear(g_aNodes[CONNECTED]);

	new count
	for(new i; i < aSize; i++){
		ArrayGetArray(g_aNodes[MAIN], i, l_Array)

		arrayset(l_Array3, -1, MAX_CONNECTABLE_NODES)
		count = 0

		for(new j; j < aSize; j++){
			if(i==j)
				continue;

			if(count == MAX_CONNECTABLE_NODES)
				break;

			ArrayGetArray(g_aNodes[MAIN], j, l_Array2)

			if((IsPosInRectangle(l_Array2[CORNER_0], l_Array[CORNER_0], l_Array[CORNER_1]))
			||(IsPosInRectangle(l_Array2[CORNER_1], l_Array[CORNER_0], l_Array[CORNER_1]))
			||(IsPosInRectangle(l_Array2[CORNER_2], l_Array[CORNER_0], l_Array[CORNER_1]))
			||(IsPosInRectangle(l_Array2[CORNER_3], l_Array[CORNER_0], l_Array[CORNER_1]))
			||(IsPosInRectangle(l_Array2[CORNER_4], l_Array[CORNER_0], l_Array[CORNER_1]))) l_Array3[++count-1] = j

			else if((IsPosInRectangle(l_Array[CORNER_0], l_Array2[CORNER_0], l_Array2[CORNER_1]))
			||(IsPosInRectangle(l_Array[CORNER_1], l_Array2[CORNER_0], l_Array2[CORNER_1]))
			||(IsPosInRectangle(l_Array[CORNER_2], l_Array2[CORNER_0], l_Array2[CORNER_1]))
			||(IsPosInRectangle(l_Array[CORNER_3], l_Array2[CORNER_0], l_Array2[CORNER_1]))
			||(IsPosInRectangle(l_Array[CORNER_4], l_Array2[CORNER_0], l_Array2[CORNER_1]))) l_Array3[++count-1] = j
		}

		ArrayPushArray(g_aNodes[CONNECTED], l_Array3)
	}
}

GetNodeNetwork(Node){
	static l_Array[MAX_CONNECTABLE_NODES], l_Array2[Items]

	ArrayGetArray(g_aNodes[CONNECTED], Node, l_Array)

	for(new i; i < MAX_CONNECTABLE_NODES; i++){
		if(l_Array[i] == -1){
			continue;
		}
		ArrayGetArray(g_aNodes[MAIN], l_Array[i], l_Array2)
		HighlightNode(l_Array[i])
	}

	#if defined DEVMODE
		client_print_color(0,0, "Node: %d | Connected: %d, %d, %d, %d", Node, l_Array[0], l_Array[1], l_Array[2], l_Array[3])
	#endif
}

Array:FindPathToNode(From, To){
	static Array:Path; Path = ArrayCreate()
	if(From < 0 || To < 0){
		log_to_file("Bad_Path.log", "A plugin nem tudott útvonalat találni 2 node között mert az egyik nem létezik. (#id < 0)")
		return Path
	}

	#if defined DEVMODE
		static Float:time; time = get_gametime()
	#endif

	static l_OpenList[MAX_NODES], l_OpenList_Count, i
	static l_CloseList[MAX_NODES], l_CloseList_Count
	static l_ListFather[MAX_NODES], Float:l_ListG[MAX_NODES], Float:l_ListF[MAX_NODES]

	l_OpenList_Count = 0
	l_CloseList_Count = 0

	for(i = 0; i < MAX_NODES; i++){
		l_ListFather[i] = -1
		l_ListF[i] = 0.0
		l_ListG[i] = 0.0
		l_OpenList[i] = 0
		l_CloseList[i] = 0
	}

	ASTAR_Add_To_List(From, l_OpenList, l_OpenList_Count)

	static Found, CurrentNode, Temp; Found = 1, CurrentNode = From, Temp = 0

	while(CurrentNode != To){
		ASTAR_Add_To_List(CurrentNode, l_CloseList, l_CloseList_Count)

		ASTAR_Remove_OpenList(CurrentNode, l_OpenList, l_OpenList_Count)

		ASTAR_HandleCost(CurrentNode, To, l_OpenList, l_OpenList_Count, l_CloseList, l_CloseList_Count, l_ListFather, l_ListG, l_ListF)

		if((Temp = ASTAR_Get_Less_F(To, l_OpenList, l_OpenList_Count, l_ListF)) == -1){
			Found = 0
			break;
		} else CurrentNode = Temp
	}

	if(!Found){
		log_to_file("Bad_Path.log", "A plugin nem tudott találni útvonalat #%d és #%d node közt.", From, To)
		return Path
	}

	ASTAR_ReversePath(CurrentNode, l_ListFather, From, Path)

	#if defined DEVMODE
		client_print_color(0,0, "^1[^4ASTAR^1] ^1Útvonal megtervezve. ^4| ^1Időtartam: ^3%f", time-get_gametime())
	#endif

	new Bad
	return FindBestWays(Path, Bad)
}

ASTAR_ReversePath(CurrentNode, l_ListFather[], From, Array:Path){
	static N; N = CurrentNode
	static Temp_Result_Count; Temp_Result_Count = 0
	
	static Reverse[MAX_NODES], Result[MAX_NODES], Result_Count

	while(l_ListFather[N] != -1)
	{
		Reverse[Temp_Result_Count++] = N
		N = l_ListFather[N]
	}
	Reverse[Temp_Result_Count++] = From

	for(new i = Temp_Result_Count-1; i > -1; i--)
	{
		Result[++Result_Count-1] = Reverse[i]
		ArrayPushCell(Path, Result[Result_Count-1])
	}
	
	Result_Count = Temp_Result_Count
}

ASTAR_Add_To_List(CurrentNode, l_List[], &l_List_Count)
	l_List[l_List_Count++] = CurrentNode;

ASTAR_Is_In_List(Node, l_List[], &l_List_Count){
	for(new i; i < l_List_Count; i++)
		if(Node == l_List[i])
			return 1
	
	return 0
}

ASTAR_Remove_OpenList(Node, l_OpenList[], &l_OpenList_Count){
	if(!l_OpenList_Count) return -1
	
	static i, x
	for(i = 0; i < l_OpenList_Count; i++)
	{
		if(Node == l_OpenList[i])
		{
			if(i < (--l_OpenList_Count))
				for(x = i; x < l_OpenList_Count; x++)
					l_OpenList[x] = l_OpenList[x+1]
			break
		}
	}
	
	return i
}

Float:Get_NodeDistance(NodeA, NodeB){
	new Float:mid1[3], Float:mid2[3]

	ArrayGetArray(g_aNodes[MIDPOINT], NodeA, mid1)
	ArrayGetArray(g_aNodes[MIDPOINT], NodeB, mid2)

	return xs_vec_distance(mid1, mid2) 
}

ASTAR_Get_Less_F(Node, l_OpenList[], l_OpenList_Count, Float:l_ListF[]){
	if(!l_OpenList_Count)
		return -1
		
	static min_n, Float:min_f
	min_n = l_OpenList[0], min_f = l_ListF[l_OpenList[0]]
	
	for(new i = 0; i < l_OpenList_Count; i++)
	{
		if(l_OpenList[i] == Node)
			return l_OpenList[i]
		
		if(min_f > l_ListF[l_OpenList[i]])
		{
			min_n = l_OpenList[i]
			min_f = l_ListF[l_OpenList[i]]
		}
	}
	
	return min_n
}

ASTAR_HandleCost(NodeA, NodeB, l_OpenList[], &l_OpenList_Count, l_CloseList[], &l_CloseList_Count, l_ListFather[], Float:l_ListG[], Float:l_ListF[]){
	static aNodes[MAX_CONNECTABLE_NODES]
	ArrayGetArray(g_aNodes[CONNECTED], NodeA, aNodes);
	for(new i = 0, neighboor; i < MAX_CONNECTABLE_NODES; i++)
	{
		if(aNodes[i] == -1)
			continue
		
		neighboor = aNodes[i]
		
		if(ASTAR_Is_In_List(neighboor, l_CloseList, l_CloseList_Count)) continue
		else {
			if(!ASTAR_Is_In_List(neighboor, l_OpenList, l_OpenList_Count))
			{
				l_ListFather[neighboor] = NodeA
				l_ListG[neighboor] = l_ListG[NodeA] + Get_NodeDistance(NodeA, neighboor);
				
				ASTAR_Set_F(neighboor, Get_NodeDistance(neighboor, NodeB), l_ListF, l_ListG)
				
				l_OpenList[l_OpenList_Count++] = neighboor
			} else {
				if(l_ListG[neighboor] < l_ListG[NodeA])
				{
					l_ListFather[neighboor] = NodeA
					l_ListG[neighboor] = l_ListG[NodeA] + Get_NodeDistance(NodeA, neighboor)
					
					l_OpenList[l_OpenList_Count++] = neighboor
				} else continue
			}
		}
	}
}

ASTAR_Set_F(Node, Float:H, Float:l_ListF[], Float:l_ListG[])
	l_ListF[Node] = l_ListG[Node] + H;

Array:FindBestWays(Array:TempArray, &Bad){
	static aSize, Float:Points[3], l_Array[Items], l_Array2[Items], i, Array:Way
	aSize = ArraySize(TempArray)

	Way = ArrayCreate(4)

	static Float:lowest1, Float:lowest2, Float:l_All[4], rip

	for(i = 0; i < aSize-1; i++){
		ArrayGetArray(g_aNodes[MAIN], ArrayGetCell(TempArray, i), l_Array)
		ArrayGetArray(g_aNodes[MAIN], ArrayGetCell(TempArray, i+1), l_Array2)

		rip = 0
		lowest2 = l_Array2[CORNER_0][2] > l_Array2[CORNER_1][2] ? l_Array2[CORNER_1][2] : l_Array2[CORNER_0][2]
		lowest1 = l_Array[CORNER_0][2] > l_Array[CORNER_1][2] ? l_Array[CORNER_1][2] : l_Array[CORNER_0][2]

		if(IsPosInRectangle(l_Array2[CORNER_0], l_Array[CORNER_0], l_Array[CORNER_1])) {
			xs_vec_copy(l_Array2[CORNER_0], Points);  Points[2] = lowest2
		}
		else if(IsPosInRectangle(l_Array2[CORNER_1], l_Array[CORNER_0], l_Array[CORNER_1])){
			xs_vec_copy(l_Array2[CORNER_1], Points);  Points[2] = lowest2
		}
		else if(IsPosInRectangle(l_Array2[CORNER_2], l_Array[CORNER_0], l_Array[CORNER_1])){
			xs_vec_copy(l_Array2[CORNER_2], Points);  Points[2] = lowest2
		}
		else if(IsPosInRectangle(l_Array2[CORNER_3], l_Array[CORNER_0], l_Array[CORNER_1])){
			xs_vec_copy(l_Array2[CORNER_3], Points);  Points[2] = lowest2
		}
		else if(IsPosInRectangle(l_Array2[CORNER_4], l_Array[CORNER_0], l_Array[CORNER_1])){
			xs_vec_copy(l_Array2[CORNER_4], Points);  Points[2] = lowest2
		}
		else if(IsPosInRectangle(l_Array[CORNER_0], l_Array2[CORNER_0], l_Array2[CORNER_1])){
			xs_vec_copy(l_Array[CORNER_0], Points);  Points[2] = lowest1
		}
		else if(IsPosInRectangle(l_Array[CORNER_1], l_Array2[CORNER_0], l_Array2[CORNER_1])){
			xs_vec_copy(l_Array[CORNER_1], Points);  Points[2] = lowest1
		}
		else if(IsPosInRectangle(l_Array[CORNER_2], l_Array2[CORNER_0], l_Array2[CORNER_1])){
			xs_vec_copy(l_Array[CORNER_2], Points);  Points[2] = lowest1
		}
		else if(IsPosInRectangle(l_Array[CORNER_3], l_Array2[CORNER_0], l_Array2[CORNER_1])){
			xs_vec_copy(l_Array[CORNER_3], Points);  Points[2] = lowest1
		}
		else if(IsPosInRectangle(l_Array[CORNER_4], l_Array2[CORNER_0], l_Array2[CORNER_1])){
			xs_vec_copy(l_Array[CORNER_4], Points);  Points[2] = lowest1
		}
		else rip = ArrayGetCell(TempArray, i)

		if(rip){
			Bad = rip
			break;
		}

		#if defined DEVMODE
			log_amx("what: %d | rip: %d | p0: %.2f | p1: %.2f | p2: %.2f", ArrayGetCell(TempArray, i), rip, Points[0], Points[1], Points[2])
		#endif

		xs_vec_copy(Points, l_All)
		l_All[3] = float(ArrayGetCell(g_aNodes[ACTION], ArrayGetCell(TempArray, i)))
		ArrayPushArray(Way, l_All)
	}

	#if defined DEVMODE
		static Float:Points2[3]
		for(i = 0; i < ArraySize(Way)-1; i++){
			ArrayGetArray(Way, i, Points)
			xs_vec_copy(Points, Points2); Points2[2] += 50.0
			exbeam(Points, Points2, 0, 0, 255)

			ArrayGetArray(Way, i+1, Points2)
			exbeam(Points, Points2, 0, 255, 0)

			xs_vec_copy(Points2, Points); Points[2] += 50.0
			exbeam(Points2, Points, 0, 0, 255)
		}
	#endif

	return Way
}

IsPosInRectangle(Float:Origin[3], Float:pt1[3], Float:pt2[3]){
	new Float:Lowest[3], Float:Highest[3]
	for(new i; i < 3; i++){
		Lowest[i] = pt1[i] < pt2[i] ? pt1[i] : pt2[i]
		Highest[i] = pt1[i] < pt2[i] ? pt2[i] : pt1[i]
	}
	remove_places(Lowest); remove_places(Highest); remove_places(Origin)

	return Origin[0] >= Lowest[0] && Origin[0] <= Highest[0] && Origin[1] >= Lowest[1] && Origin[1] <= Highest[1] && Origin[2] >= Lowest[2] && Origin[2] <= Highest[2]
}

HighlightNode(Node, life = 7){
	new l_Array[Items], l_Type
	switch(Node){
		case -1: return;
		case -2: {
			for(new i; i < ArraySize(g_aNodes[MAIN]); i++){
				ArrayGetArray(g_aNodes[MAIN], i, l_Array)
				l_Type = ArrayGetCell(g_aNodes[ACTION], i)

				DrawRectangle(l_Array[CORNER_0], l_Array[CORNER_1], life, g_cActionColors[l_Type][0], g_cActionColors[l_Type][1], g_cActionColors[l_Type][2])
			}
		}
		default:{
			ArrayGetArray(g_aNodes[MAIN], Node, l_Array)
			l_Type = ArrayGetCell(g_aNodes[ACTION], Node)

			DrawRectangle(l_Array[CORNER_0], l_Array[CORNER_1], life, g_cActionColors[l_Type][0], g_cActionColors[l_Type][1], g_cActionColors[l_Type][2])
		}
	}
}

ClearBits(){
	ClearBit(g_bits, BIT_TRACK_EXISTING_POINTS); ClearBit(g_bits, BIT_NEW_POINT_SET1); ClearBit(g_bits, BIT_NEW_POINT_SET2);
	ClearBit(g_bits, BIT_POINT_SET1); ClearBit(g_bits, BIT_POINT_SET2)
	g_iNode = 0;
	g_iActionType = 0;
}

public ShowBeam(iId){
	if(!GetBit(g_bits, BIT_TRACK_EXISTING_POINTS)){
		remove_task(iId)
		return;
	}

	new inp;
	if((inp = GetNewPoint()) != -1){
		new Float:aim[3]; fm_get_aim_origin(iId, aim)
		exbeam(g_iPoints[inp], aim, 0, 255, 0, 1)
		exbeam(g_iPoints[!inp ? 1 : 0], aim, 0, 0, 255, 1)
		DrawRectangle(g_iPoints[!inp ? 1 : 0], aim, 1, g_cActionColors[g_iActionType][0], g_cActionColors[g_iActionType][1], g_cActionColors[g_iActionType][2])
		return;
	}

	if(!g_iPoints[1]){
		new Float:o[3]; pev(iId, pev_origin, o)
		exbeam(g_iPoints[0], o, 0, 0, 255, 1)
	} else exbeam(g_iPoints[0], g_iPoints[1], 0, 0, 255, 1)
}

GetNodeByOrigin(iId){
	if(g_Nodes != 0){
		new Float:aim[3]
		pev(iId, pev_origin, aim)
		new l_Array[Items]
		for(new i; i < g_Nodes; i++){
			ArrayGetArray(g_aNodes[MAIN], i, l_Array)

			if(IsPosInRectangle(aim, l_Array[CORNER_0], l_Array[CORNER_1])){
				#if defined DEVMODE
					HighlightNode(i)
				#endif
				return i;
			}
		}
	}
	return -1
}

GetNodeByAim(iId){
	new Float:aim[3]; fm_get_aim_origin(iId, aim)
	new l_Array[Items]
	for(new i; i < g_Nodes; i++){
		ArrayGetArray(g_aNodes[MAIN], i, l_Array)

		if(IsPosInRectangle(aim, l_Array[CORNER_0], l_Array[CORNER_1])){
			HighlightNode(i)
			return i;
		}
	}
	return -1
}

Modify_Node(iId){
	new iNode, l_Array[Items]
	if((iNode = GetNodeByAim(iId)) == -1){
		Menu_Node(iId)
		return;
	}

	ArrayGetArray(g_aNodes[MAIN], iNode, l_Array)

	g_iActionType = ArrayGetCell(g_aNodes[ACTION], iNode)

	xs_vec_copy(l_Array[CORNER_0], g_iPoints[0])
	xs_vec_copy(l_Array[CORNER_1], g_iPoints[1])

	SetBit(g_bits, BIT_POINT_SET2);
	SetBit(g_bits, BIT_POINT_SET1);

	g_iNode = iNode+1

	Menu_Create_Node(iId)
}

GetNewPoint(){
	if(GetBit(g_bits, BIT_NEW_POINT_SET1))
		return 0
	else if(GetBit(g_bits, BIT_NEW_POINT_SET2))
		return 1
	return -1
}

PickUpViewingPoint(iId){
	new Float:aim[3]; fm_get_aim_origin(iId, aim)
	new Float:nPoints[Items], item = -1

	CheckEquality(g_iPoints[0], g_iPoints[1], aim, 100.0, item)

	if(item != -1){
		SetBit(g_bits, BIT_TRACK_EXISTING_POINTS)

		if(nPoints[item])
			xs_vec_copy(nPoints[item], g_iPoints[item])

		if(!item)
			SetBit(g_bits, BIT_NEW_POINT_SET1)
		else
			SetBit(g_bits, BIT_NEW_POINT_SET2)
		if(!task_exists(iId))
			set_task(0.1, "ShowBeam", iId, .flags = "b")

		Menu_Create_Node(iId)
	}

	return PLUGIN_HANDLED;
}

CheckEquality(Float:p1v1[3], Float:p1v2[3], Float:p2[3], Float:fDiff, &item){
	if(xs_vec_unique_equal(p1v1, p2, fDiff)){
		item = 0;
	}

	if(xs_vec_unique_equal(p1v2, p2, fDiff)){
		item = 1
		if(!item){
			if(xs_vec_distance(p1v1, p2) < xs_vec_distance(p1v2, p2))
				item = 0
			else
				item = 1
		}
	}
	if(item != -1) xs_vec_copy(!item ? p1v1 : p1v2, g_iPoints[item])
}

GetMidpoint(Float:fOrigin[3], Float:fOrigin2[3], Float:fOutPut[3]){
	for(new i; i < 3; i++) fOutPut[i] = (fOrigin[i] + fOrigin2[i]) / 2.0
}

DrawRectangle(Float:Origin[3], Float:Origin2[3], life = 101, r = 255, g = 0, b = 0){
	new Float:Highest, Float:Lowest
	Highest = Origin[2] < Origin2[2] ? Origin2[2] : Origin[2]
	Lowest = Origin[2] < Origin2[2] ? Origin[2] : Origin2[2]

	new Float:p1[3]; xs_vec_copy(Origin, p1); new Float:p2[3]; xs_vec_copy(Origin2, p2)

	new Float:vertex[4][3];

	#define A 0
	#define B 1
	#define C 2
	#define D 3

	p1[2] = Lowest; p2[2] = Lowest;

	xs_vec_copy(p1, vertex[A])
	xs_vec_copy(p2, vertex[C])
	p2[0] = p1[0]; xs_vec_copy(p2, vertex[B])
	p1[0] = Origin2[0]; xs_vec_copy(p1, vertex[D])

	exbeam(vertex[A], vertex[D], r, g, b, life)
	exbeam(vertex[A], vertex[B], r, g, b, life)
	exbeam(vertex[C], vertex[B], r, g, b, life)
	exbeam(vertex[C], vertex[D], r, g, b, life)

	new Float:vertexup[4][3];

	for(new i; i < 4; i++){
		xs_vec_copy(vertex[i], vertexup[i])
		vertexup[i][2] = Highest;
		exbeam(vertex[i], vertexup[i], r, g, b, life)
	}

	xs_vec_copy(Origin, p1); xs_vec_copy(Origin2, p2)

	p1[2] = Highest; p2[2] = Highest;

	xs_vec_copy(p1, vertex[A])
	xs_vec_copy(p2, vertex[C])
	p2[0] = p1[0]; xs_vec_copy(p2, vertex[B])
	p1[0] = Origin2[0]; xs_vec_copy(p1, vertex[D])

	exbeam(vertex[A], vertex[D], r, g, b, life)
	exbeam(vertex[A], vertex[B], r, g, b, life)
	exbeam(vertex[C], vertex[B], r, g, b, life)
	exbeam(vertex[C], vertex[D], r, g, b, life)
}

xs_vec_unique_equal(Float:vec1[3], Float:vec2[3], Float:Diff){
	if(xs_vec_distance(vec1, vec2) <= Diff){
		return 1
	}
	return 0;
}

exbeam(Float:iorigin[3], Float:forigin[3], r, g, b, life = 202){
	message_begin(MSG_BROADCAST, SVC_TEMPENTITY)
	write_byte(TE_BEAMPOINTS)
	for(new k; k < 3; k++) write_coord_f(iorigin[k])
	for(new k; k < 3; k++) write_coord_f(forigin[k])
	write_short(g_dot)
	write_byte(0); write_byte(1); write_byte(life); write_byte(5); write_byte(0)
	write_byte(r); write_byte(g); write_byte(b); write_byte(200); write_byte(0)
	message_end()
}

remove_places(Float:vec[3]){
	vec[0] = float(floatround(vec[0]))
	vec[1] = float(floatround(vec[1]))
	vec[2] = float(floatround(vec[2]))
}