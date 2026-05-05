//+------------------------------------------------------------------+
//| Onasis Ichimoku Always In Market EA                              |
//+------------------------------------------------------------------+
#property strict

input int Tenkan = 9;
input int Kijun  = 26;
input int SenkouB = 52;

input double Lots = 0.1;
input int Slippage = 3;
input int Magic = 2026;

//+------------------------------------------------------------------+
// FUNCIONES ICHIMOKU
//+------------------------------------------------------------------+
double GetTenkan(int shift)
{
   double high = iHigh(NULL,0,iHighest(NULL,0,MODE_HIGH,Tenkan,shift));
   double low  = iLow(NULL,0,iLowest(NULL,0,MODE_LOW,Tenkan,shift));
   return (high + low)/2.0;
}

double GetKijun(int shift)
{
   double high = iHigh(NULL,0,iHighest(NULL,0,MODE_HIGH,Kijun,shift));
   double low  = iLow(NULL,0,iLowest(NULL,0,MODE_LOW,Kijun,shift));
   return (high + low)/2.0;
}

double GetSenkouB(int shift)
{
   double high = iHigh(NULL,0,iHighest(NULL,0,MODE_HIGH,SenkouB,shift));
   double low  = iLow(NULL,0,iLowest(NULL,0,MODE_LOW,SenkouB,shift));
   return (high + low)/2.0;
}

//+------------------------------------------------------------------+
// DETECTAR POSICIÓN ACTUAL
//+------------------------------------------------------------------+
int CurrentPosition()
{
   for(int i=0;i<OrdersTotal();i++)
   {
      if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES))
      {
         if(OrderMagicNumber()==Magic)
         {
            if(OrderType()==OP_BUY) return 1;
            if(OrderType()==OP_SELL) return -1;
         }
      }
   }
   return 0;
}

//+------------------------------------------------------------------+
// CERRAR TODAS
//+------------------------------------------------------------------+
void CloseAll()
{
   for(int i=OrdersTotal()-1;i>=0;i--)
   {
      if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES))
      {
         if(OrderMagicNumber()==Magic)
         {
            if(OrderType()==OP_BUY)
               OrderClose(OrderTicket(),OrderLots(),Bid,Slippage);
            if(OrderType()==OP_SELL)
               OrderClose(OrderTicket(),OrderLots(),Ask,Slippage);
         }
      }
   }
}

//+------------------------------------------------------------------+
// ON TICK
//+------------------------------------------------------------------+
void OnTick()
{
   static datetime lastBar = 0;
   if(Time[0] == lastBar) return;
   lastBar = Time[0];

   // ICHIMOKU
   double tenkan0 = GetTenkan(0);
   double kijun0  = GetKijun(0);
   double tenkan1 = GetTenkan(1);
   double kijun1  = GetKijun(1);

   // CRUCES
   bool bullCross = (tenkan1 < kijun1 && tenkan0 > kijun0);
   bool bearCross = (tenkan1 > kijun1 && tenkan0 < kijun0);

   // NUBE
   double senkouA = (tenkan0 + kijun0)/2.0;
   double senkouB = GetSenkouB(0);

   double topKumo = MathMax(senkouA, senkouB);
   double botKumo = MathMin(senkouA, senkouB);

   bool bullKumo = Close[1] > topKumo;
   bool bearKumo = Close[1] < botKumo;

   // CHIKOU (simplificado)
   bool chikouBull = Close[26] > High[26];
   bool chikouBear = Close[26] < Low[26];

   // CONDICIONES
   bool longCond  = bullCross && bullKumo && chikouBull;
   bool shortCond = bearCross && bearKumo && chikouBear;

   int pos = CurrentPosition();

   // =========================
   // LÓGICA ALWAYS IN
   // =========================

   // NO HAY POSICIÓN → ABRIR
   if(pos == 0)
   {
      if(longCond)
         OrderSend(Symbol(),OP_BUY,Lots,Ask,Slippage,0,0,"Buy",Magic,0,clrGreen);

      if(shortCond)
         OrderSend(Symbol(),OP_SELL,Lots,Bid,Slippage,0,0,"Sell",Magic,0,clrRed);
   }

   // SI HAY BUY → REVERSAR A SELL
   if(pos == 1 && shortCond)
   {
      CloseAll();
      OrderSend(Symbol(),OP_SELL,Lots,Bid,Slippage,0,0,"Sell",Magic,0,clrRed);
   }

   // SI HAY SELL → REVERSAR A BUY
   if(pos == -1 && longCond)
   {
      CloseAll();
      OrderSend(Symbol(),OP_BUY,Lots,Ask,Slippage,0,0,"Buy",Magic,0,clrGreen);
   }
}
