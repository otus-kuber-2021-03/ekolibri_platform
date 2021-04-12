FROM nginx
RUN mkdir /app
RUN usermod -G nginx,root -u 1001 nginx 
COPY index.html /app/ 
COPY nginx.conf /etc/nginx/nginx.conf 
RUN chown -R nginx /etc/nginx && chown -R nginx /app
EXPOSE 8000
ENTRYPOINT ["nginx", "-g", "daemon off;"]
