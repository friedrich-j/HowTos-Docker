FROM debian AS stage1
ENV DEBIAN_FRONTEND=noninteractive LAYER=stage1
RUN touch /tmp/stage1.txt && echo -e "\e[91m1: not cached\e[0m"

FROM tmp/multistage/03:stage_stage1 AS stage2
ENV DEBIAN_FRONTEND=noninteractive LAYER=stage2
RUN touch /tmp/stage2.txt && echo -e "\e[91m2: not cached\e[0m"

FROM debian
ENV DEBIAN_FRONTEND=noninteractive LAYER=final
COPY --from=stage1 /tmp/* /tmp/
COPY --from=stage2 /tmp/* /tmp/
RUN ls -l /tmp/ && echo -e "\e[91m3: not cached\e[0m"
