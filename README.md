//+------------------------------------------------------------------+
//| Onasis Ichimoku Flow EA (Always In)                              |
//+------------------------------------------------------------------+
#property strict

input int Tenkan = 9;
input int Kijun  = 26;

input double Lots = 0.1;
input int Slippage = 3;
input int Magic = 777;

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

//+------------------------------------------------------------------+
// POSICIÓN ACTUAL
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
         if(OrderMagicNumber()!=Magic) continue;

         if(OrderType()==OP_BUY)
            OrderClose(OrderTicket(),OrderLots(),Bid,Slippage);

         if(OrderType()==OP_SELL)
            OrderClose(OrderTicket(),OrderLots(),Ask,Slippage);
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

   // =========================
   // DATOS
   // =========================
   double tenkan0 = GetTenkan(0);
   double kijun0  = GetKijun(0);

   double open1  = Open[1];
   double close1 = Close[1];

   int pos = CurrentPosition();

   // =========================
   // 1. CONDICIÓN DE INICIO
   // =========================
   bool startBull = (open1 < tenkan0 && close1 > tenkan0);

   // =========================
   // 2. ESTADO DE MERCADO
   // =========================
   bool bullish = (tenkan0 > kijun0 && Close[1] > tenkan0);
   bool bearish = (tenkan0 < kijun0 && Close[1] < tenkan0);

   // =========================
   // INICIO DEL SISTEMA
   // =========================
   if(pos == 0)
   {
      if(startBull)
      {
         OrderSend(Symbol(),OP_BUY,Lots,Ask,Slippage,0,0,"Start Buy",Magic,0,clrGreen);
         return;
      }
   }

   // =========================
   // ALWAYS-IN (REVERSIÓN)
   // =========================

   // Si está en BUY y aparece BEARISH → cambia a SELL
   if(pos == 1 && bearish)
   {
      CloseAll();
      OrderSend(Symbol(),OP_SELL,Lots,Bid,Slippage,0,0,"Sell",Magic,0,clrRed);
      return;
   }

   // Si está en SELL y aparece BULLISH → cambia a BUY
   if(pos == -1 && bullish)
   {
      CloseAll();
      OrderSend(Symbol(),OP_BUY,Lots,Ask,Slippage,0,0,"Buy",Magic,0,clrGreen);
      return;
   }
}
