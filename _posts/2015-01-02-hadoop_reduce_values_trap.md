---
title: Traps in hadoop reducer's values parameter
layout: post
---

To learn hadoop, I wrote a simple application to calculate min price and max price of all NYSE stocks in a period.

{% highlight java linenos tabsize=4 %}
public class MinMaxCalc {

    public static class StockMinMaxWritable implements  Writable {
        private DateWritable minDate;
        private HiveDecimalWritable minPrice;
        private DateWritable maxDate;
        private HiveDecimalWritable maxPrice;

        public StockMinMaxWritable() {
            minDate = new DateWritable();
            minPrice = new HiveDecimalWritable();
            maxDate = new DateWritable();
            maxPrice = new HiveDecimalWritable();
        }

        public DateWritable getMinDate() {
            return minDate;
        }

        public void setMinDate(DateWritable minDate) {
            this.minDate = minDate;
        }

        public HiveDecimalWritable getMinPrice() {
            return minPrice;
        }

        public void setMinPrice(HiveDecimalWritable minPrice) {
            this.minPrice = minPrice;
        }

        public DateWritable getMaxDate() {
            return maxDate;
        }

        public void setMaxDate(DateWritable maxDate) {
            this.maxDate = maxDate;
        }

        public HiveDecimalWritable getMaxPrice() {
            return maxPrice;
        }

        public void setMaxPrice(HiveDecimalWritable maxPrice) {
            this.maxPrice = maxPrice;
        }

        @Override
        public void write(DataOutput out) throws IOException {
            minDate.write(out);
            minPrice.write(out);
            maxDate.write(out);
            maxPrice.write(out);
        }

        @Override
        public void readFields(DataInput in) throws IOException {
            minDate.readFields(in);
            minPrice.readFields(in);
            maxDate.readFields(in);
            maxPrice.readFields(in);
        }

        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();
            sb.append(minDate.toString()).append('\t').
                    append(minPrice.toString()).append('\t').
                    append(maxDate.toString()).append('\t').
                    append(maxPrice.toString());
            return sb.toString();
        }
    }

    public static class StockDailyDataWritable implements Writable {
        private DateWritable date;
        private HiveDecimalWritable priceOpen;
        private HiveDecimalWritable priceHigh;
        private HiveDecimalWritable priceLow;
        private HiveDecimalWritable priceClose;

        public StockDailyDataWritable() {
            date = new DateWritable();
            priceOpen = new HiveDecimalWritable();
            priceHigh = new HiveDecimalWritable();
            priceLow = new HiveDecimalWritable();
            priceClose = new HiveDecimalWritable();
        }

        public DateWritable getDate() {
            return date;
        }

        public void setDate(DateWritable date) {
            this.date = date;
        }

        public HiveDecimalWritable getPriceOpen() {
            return priceOpen;
        }

        public void setPriceOpen(HiveDecimalWritable priceOpen) {
            this.priceOpen = priceOpen;
        }

        public HiveDecimalWritable getPriceHigh() {
            return priceHigh;
        }

        public void setPriceHigh(HiveDecimalWritable priceHigh) {
            this.priceHigh = priceHigh;
        }

        public HiveDecimalWritable getPriceLow() {
            return priceLow;
        }

        public void setPriceLow(HiveDecimalWritable priceLow) {
            this.priceLow = priceLow;
        }

        public HiveDecimalWritable getPriceClose() {
            return priceClose;
        }

        public void setPriceClose(HiveDecimalWritable priceClose) {
            this.priceClose = priceClose;
        }

        @Override
        public void write(DataOutput out) throws IOException {
            date.write(out);
            priceOpen.write(out);
            priceHigh.write(out);
            priceLow.write(out);
            priceClose.write(out);
        }

        @Override
        public void readFields(DataInput in) throws IOException {
            if (priceClose == null) priceClose = new HiveDecimalWritable();
            date.readFields(in);
            priceOpen.readFields(in);
            priceHigh.readFields(in);
            priceLow.readFields(in);
            priceClose.readFields(in);
        }
    }

    public static class StockMapper extends Mapper<Object, Text, Text, StockDailyDataWritable> {
        @Override
        protected void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            //super.map(key, value, context);
            String[] fields = value.toString().split("\t");
            StockDailyDataWritable writable = new StockDailyDataWritable();
            SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
            Date date = null;
            try {
                date = new Date(dateFormat.parse(fields[2]).getTime());
            } catch (ParseException e) {
                e.printStackTrace();
            }
            writable.setDate(new DateWritable(date));
            writable.setPriceOpen(new HiveDecimalWritable(HiveDecimal.create(new BigDecimal(fields[3]))));
            writable.setPriceHigh(new HiveDecimalWritable(HiveDecimal.create(new BigDecimal(fields[4]))));
            writable.setPriceLow(new HiveDecimalWritable(HiveDecimal.create(new BigDecimal(fields[5]))));
            writable.setPriceClose(new HiveDecimalWritable(HiveDecimal.create(new BigDecimal(fields[6]))));
            Text symbol = new Text(fields[1]);
            context.write(symbol, writable);
        }
    }

    public static class StockReducer extends Reducer<Text, StockDailyDataWritable,Text, StockMinMaxWritable> {

        @Override
        protected void reduce(Text key, Iterable<StockDailyDataWritable> values, Context context) throws IOException, InterruptedException {
            StockMinMaxWritable result = null;
            for (StockDailyDataWritable day: values) {
                if(null == result) {
                    result = new StockMinMaxWritable();
                    result.setMinDate(day.getDate());
                    result.setMinPrice(day.getPriceLow());
                    result.setMaxDate(day.getDate());
                    result.setMaxPrice(day.getPriceHigh());
                }
                else {
                    if (day.getPriceLow().compareTo(result.getMinPrice()) < 0 ||
                            (day.getPriceLow().compareTo(result.getMinPrice()) == 0 && day.getDate().compareTo(result.getMinDate()) < 0)) {
                        result.setMinPrice(day.getPriceLow());
                        result.setMinDate(day.getDate());
                    }
                    else if (day.getPriceHigh().compareTo(result.getMaxPrice()) >0 ||
                            (day.getPriceHigh().compareTo(result.getMaxPrice()) == 0 && day.getDate().compareTo(result.getMinDate()) > 0)) {
                        result.setMaxPrice(day.getPriceHigh());
                        result.setMaxDate(day.getDate());
                    }
                }
            }
            context.write(key, result);
        }
    }

    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "Min max calc");
        job.setJarByClass(MinMaxCalc.class);
        job.setMapperClass(StockMapper.class);
        job.setReducerClass(StockReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(StockDailyDataWritable.class);
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }

}

{% endhighlight %}

Unfortunatelly I found the result is wrong. All min price and max price are on the same day. 

{% highlight text linenos tabsize=4 %}
AA	2001-03-05	37.38	2001-03-05	38.45
AAI	2000-05-08	4.44	2000-05-08	4.75
AAN	2000-04-10	14	2000-04-10	14.69
AAP	2001-12-12	42.11	2001-12-12	43.75
AAR	2001-05-11	24.52	2001-05-11	24.65
...
{% endhighlight %}

At first I thought there may be something wrong with the input: all the input values for one key have same value. But after I dumped all the values to output, all of them are as expected. Then I found something interesting, the comparisions on line 187 and 192 always return 0. In another word, the result object's min and max price fields changed in every iteration. Who changed the value? 

After some investigation, I found that Reducer's context only created one instance of VALUEIN, and in every iteration, called its readFields methods to update the fields. So at the line 178, day's readFields methods is called implicitly. Since line 181 to 184 have set results' fields to day's internal fields, so the fields changed by line 178 in every iteration and the min/max price and date would never be updated later.
![hadoop reducer values retrival](/images/hadoop_reducer_valuein.jpg)

Then the fix is trival, just changing the set methold, using `this.minDate.set(minDate)` instead of `this.minDate = minDate`. And finally we got the expected results:

{% highlight text linenos tabsize=4 %}
AA	2000-10-20	23.12	2000-01-10	87.25
AAI	2001-09-20	2.6	2001-06-14	12.25
AAN	2000-06-29	11.47	2001-06-05	19.5
AAP	2001-12-04	39.7	2001-12-31	49.75
AAR	2001-09-21	17.6	2001-08-08	25.25
...
{% endhighlight %}
